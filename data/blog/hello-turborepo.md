---
title: Pitfalls When Adding Turborepo To Your Project
authors: [tcarre]
date: 2022-02-10
tags: [devops]
draft: false
summary: Challenges scaling monorepo with npm workspaces, turborepo and CircleCI
---

We at Caribou have recently adopted a new TypeScript monorepo stack for our app frontends using [turborepo](https://turborepo.org/).

## Issues faced with our original monorepo setup

As our number of apps and codebases grew, we decided that we wanted to:

- Save time and money on ever increasing build times
    - Build times were increasing dramatically as we went from 2 apps in our monorepo to 4. The original monorepo setup would naively deploy all apps inside the project on every push to GitHub. Once we got to 4 projects, the build times got really out of hand.
- Enable the granular tracking of individual application deployments for our metrics monitoring
    - We strive to do 5 releases per week on average (across all of our projects), and we need to track whether we're hitting these targets or not.
- Add CircleCI as a layer between GitHub and Netlify to manage our CI pipeline
    - Our other repos were already on CircleCI, so this allowed us to unify our CI/CD process.

As we’ve faced multiple roadblocks undergoing this transition, we decided to record them for the benefit of developers at Caribou or anyone else undertaking similar endeavours.

## Starting point and stack choice

![Directory structure](/static/images/monorepo/1642780105.png)

We started from a flat file system containing multiple apps that were all located in the project’s root folder. The directory structure itself needed work.

### Researching & The Design Document Phase

At Caribou, net-new functionality or highly complex additions to our systems must go through a design document process.

We wrote a design document outlining our requirements and how the new stack would meet them. Our requirements were not complicated. We wanted to rebuild and deploy only those parts of the monorepo that have changed, and add our desired checks on CircleCI. 

We had a first look at monorepo management. We knew [Lerna](https://lerna.js.org) was a popular choice, but [Turborepo](https://turborepo.org) had recently been acquired by Vercel and seemed highly promising. It purported to be very fast but simpler than Lerna, and one of our engineers had a positive experience with it.

After a few days of playing around with Turborepo, we concluded that its simple and intuitive API was sufficient justification to proceed with it as our tool of choice.

Turborepo works with one of Yarn, npm, or pnpm workspaces. We already used npm as a package manager, so in order to keep things familiar we went with npm workspaces.

Finally, we already used CircleCI for our backend CI, so we wanted to keep using
CircleCI on the frontend.

## Setting up the npm workspace

This is [easily done inside the root `package.json`](https://turborepo.org/docs/getting-started#ensure-workspaces-are-configured).

### Run `npm install` to create the symlinks inside `node_modules`

One thing to note is to not forget to rerun `npm install` at the project root (we initially did...). If you forget, npm won’t create the symlinks to your workspaces/packages inside `node_modules`, and you won’t be able to use absolute paths to other modules inside your imports. 

### npm v7 is needed or the IDE/compiler can't resolve modules

Even if you run `npm install`, only npm 7 and up support workspaces. [There is no straightforward way to enforce developer npm version](https://github.com/npm/rfcs/discussions/50) although it is not impossible, so you might want to document the version requirement in your root README. A developer without npm 7+ will end up with unresolved modules in their editor.

### New commands to install dependencies and run scripts

When using npm packages, you must keep in mind that the commands to install dependencies and run scripts are different.

Assuming a sub-package named `blog`, installing the dependency `neverthrow` is done by running this command at the monorepo root:

```bash
# DON'T do that anymore
npm install neverthrow
# Do this instead
npm install --workspace blog neverthrow
# or for short
npm i -w blog neverthrow
```

Running the `start` script from the `blog` subpackage is done with the following:

```bash
# Don't do that anymore
npm run start
# Do this instead
npm run --workspace blog start
# or for short
npm run -w blog start 
```

## Separating dependencies

One detail which was not immediately obvious during the transition is that the root `package.json` should only contain dev dependencies. (It doesn’t need to be all of them, either.) We initially thought we should keep common dependencies in the root package.json. This caused React errors from having multiple instances of React running. 

Another thing to note is you should never see a `package-lock.json` inside a sub-package’s folder. This means the `npm install` command was run inside it, which is incorrect! Delete the resulting `package-lock.json` as well as the `node_modules` it newly installed. When using npm workspaces, all dependencies live in the root `node_modules`.

## Import resolution after transitioning

We use webpack for our build pipeline, and found out that `webpack` was sometimes resolving modules that `tsc` couldn’t. This is problematic, as we wanted to use `tsc` for our CI checks! After experimentation, I found that imports must adhere to the following format:

- Absolute imports from the current package must not be prefixed with the package’s name, i.e. if you are currently inside `ha-dash` (the name of one of our sub-projects within the monorepo) you must write `import { whatever } from 'src/components` and not `import { whatever } from 'ha-dash/src/components'`.
    - The `src` may be skipped by setting that package’s `baseUrl` to `src` in its `tsconfig.json`
- Absolute imports from other packages must be written as `{package_name}/src/some_module`
    - Unfortunately we haven’t found how to skip the `/src/`  for cross-package imports yet. [This solution](https://github.com/nodejs/node/issues/14970#issuecomment-571887546) seemed promising but it causes the typescript compiler to hang for some reason.

While transitioning and changing import paths, I’ve often used Linux shell loops like the following:

```bash
# make sure your current directory is the package you wish to perform changes in
# commit your current repo state so you can undo in case of mistake!
for file in **/**.{ts,tsx}; do
  sed -i -e "s?from 'survey-manager-src/?from '?g" $file;
done
```

while in the `survey-manager` directory, I ran this command to change all instances of `from 'survey-manager-src/` to `from '`.

## Failing tests

We use `jest` for tests, and found that in order for tests to work in our setup we needed each package to contain a `babel.config.js` file including `'@babel/preset-react'`. This may be applicable to your pipeline, too!

## CircleCI

### Saving turbo cache artifacts between builds

Turborepo stores build artifacts at `node_modules/.cache` in order to restore files which do not need to be rebuilt.

```yaml
build:
    executor: caribou
    resource_class: xlarge
    steps:
      - checkout
      - attach_workspace:
          at: .
      - restore_cache:
          keys:
            - previous-build-{{ .Branch }}
      - run:
          name: "Build apps"
          command: npx turbo run build
      - save_cache:
          key: previous-build-{{ .Branch }}
          paths:
            - node_modules/.cache
      - persist_to_workspace:
          root: .
          paths:
            - apps/
```

The important sections here are `restore_cache` and `save_cache`. Basically this looks for any turborepo cache saved by CircleCI named `previous-build-{name_of_current_branch}`. Then turbo will know what packages it needs to rebuild.

The `persist_to_workspace` section is important, as it lets the next step (`deploy`) have access to the built files.

```yaml
deploy:
    executor: caribou
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: "Deploy to netlify"
          command: ./deploy.sh ${CIRCLE_BRANCH} ${CIRCLE_SHA1}
```

### Saving dependencies between builds

While you are at it, you can cache npm dependencies between runs. The strategy is slightly different:

```yaml
install-deps:
    executor: caribou
    steps:
      - checkout
      - restore_cache:
          keys:
            - npm-deps-{{ checksum "package-lock.json" }}
            - npm-deps-
      - run:
          name: "Install Dependencies"
          command: |
            echo "Node version: $(node -v)"
            echo "npm version: $(npm -v)"
            npm install
      - save_cache:
          key: npm-deps-{{ checksum "package-lock.json" }}
          paths:
            - node_modules
      - persist_to_workspace:
          root: .
          paths:
            - node_modules
```

We use `npm-deps-{{ checksum "package-lock.json" }}` this time, to look for cached node modules from runs of *any branch* that had the same `package-lock.json`. If none is found, we simply get the latest cached `node_modules`. Then `npm install` is run anyway, so that any missing package is added.

## ⚠️ The netlify CLI cannot use same URL prefixes as automatic branch deployments

[https://github.com/netlify/cli/issues/1984#issuecomment-862554734](https://github.com/netlify/cli/issues/1984#issuecomment-862554734)

If you’ve previously used automatic netlify deployments by branch, then you might be used to having URLs formatted as `{branch-name}--{site-name}.netlify.app`.

As soon as you’ve used this feature once, you can no longer use that subdomain with the Netlify CLI. We had to move to other prefixes using the Netlify CLI `--alias` option. The documentation says to “avoid” using the same prefix as branch names, but doesn’t say why... now you know! [Here is the GitHub issue about this](https://github.com/netlify/cli/issues/1984).

## Only deploying the individual apps which turbo rebuilt

This is something which the [documentation for the netlify CLI](https://cli.netlify.com/commands/deploy) doesn't tell you, so you won't find out until you actually run it: **the netlify CLI compares the newest build's file hashes with the previous build's hashes, and requests only those files which have changed.** In other words, you can safely use the netlify CLI to trigger deployments of *all* your packages, and netlify will only ever receive those files which have changed.

However, if you are using something less sophisticated than netlify, here's a bash script I wrote before I realised that netlify already took care of this. This script will parse the turbo build output and only redeploy apps which turbo deemed necessary to rebuild.

```bash
# Save the turbo output with this command:
# $ npx turbo run build 2>&1 | tee .build_output

APPS=("blog" "client-dashboard" "admin-panel")

deploy_app() {
  app_name=$1
  # your deployment command here
}

for app in ${APPS[@]}; do
  case "$(cat ./.build_output)" in
    *"${app}:build: cache miss, executing"*) deploy_app "$app" ;;
    *"${app}:build: cache bypass, force"*) deploy_app "$app" ;;
    # Uncomment the first *) line to force deployment
    # *) deploy_app "$app" ;;
    *) echo "turbo did not rebuild $app, not deploying." ;;
  esac
done

```

And for whoever it might help, our netlify deploy function:

```bash
# Those environment variables are set in CircleCI
site_id_of() {
  case "$1" in
    ha-dash) echo "$HA_DASH_NETLIFY_ID" ;;
    fa-dash) echo "$FA_DASH_NETLIFY_ID" ;;
    planner) echo "$PLANNER_NETLIFY_ID" ;;
    survey-manager) echo "$SURVEY_MANAGER_NETLIFY_ID" ;;
  esac
}

deploy_app() {
  app_name=$1
  if [ "$BRANCH" = "production" ]; then
    branch_option=""
  else
    branch_option="--alias staging-branch"
  fi
  # --prod argument applies to staging too
  npx netlify deploy \
    --auth=$NETLIFY_AUTH_TOKEN \
	  --dir=./apps/$app_name/build \
    --message="$BRANCH deployment of $GIT_HASH" \
    --prod \
    --site=$(site_id_of "$appName") \
    $branch_option
}
```

# Conclusion

Do you have experience transitioning to monorepo management tools? Do you see anything we can improve? Let us know! I hope this log of some of the challenges making the transition can be helpful to some of you. Happy hacking!

# Did you enjoy this post? We're hiring!

We have several [open roles](https://caribouwealth.notion.site/Careers-at-Caribou-357783b6007f4d15bae8c7b9651cebf5) across Ops, Design, Marketing and Engineering!

