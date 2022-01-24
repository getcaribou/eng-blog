---
title: 2021 Year Review
authors: [giorgiodelgado]
date: 2022-01-08
tags: [management]
draft: false
summary: Documenting 2021's highlights of Caribou's Engineering team
---

2021 proved to be a massive success for Caribou as a whole. We managed to:

- Form a fast-paced and experienced founding engineering team with an obsession over code quality + correctness, functional programming and moving hella fast ⚡
- Close 7 Firms, each with somewhere between \$300 million to \$2.7B in Assets Under Management!!!
- Ship 3 separate web applications to end users, develop an email notifications system, a feature flagging system, a secure HIPAA-compliant email alternative, Firm management system, and much more 
- Closed our seed round of $2.5M by some of the best angels in the startup scene (round led by the Altmans)

In this post I will be going over some of our proudest technical moments.

## First order of business: Getting Our First App In Production

After quite a lot of back-and-forth, a search for a founding engineer had come to its conclusion in January. [I joined Caribou](https://www.linkedin.com/in/giorgiodelgado/) with a burning desire to execute on the company's vision of *creating a world where healthcare doesn't get in the way of living*, in addition to also executing on my own passion for creating high-performance & exceptional engineering teams.

At the time of my joining, Caribou had a single web application (codenamed "**HA Dash**") that had been developed by a contractor in a waterfall style.

Given the waterfall-style of **HA Dash**'s development, it had not been launched despite it already having a lot of useful functionality already.

![early HA Dash](/static/images/early-ha-dash.png)

*Screenshot of one of the pages of "HA Dash", which we use to manage cases (anything from insurance cost analysis to urgent cancer healthcare navigation and planning).*

Thus, it became clear that the first order of business for the engineering team and culture would be to develop a mindset of "**always be shipping**". And so I went about setting up the infrastructure to allow for such a culture to flourish:

- Transition over to GCP
- Move our API server over to GCP App Engine for hassle-free / serveless application management
- Set up a production MySQL database using GCP Cloud SQL
- Create a staging and production environment on GCP
- Set up CI/CD to allow for a quick develop, test, deploy cycle

After about 2 weeks of setting up all of this scaffolding, we were in a good place to finally ship **HA Dash** to production. This allowed our staff to begin doing Case Management on our systems as opposed to on Google Sheets.

Looking back, these decisions proved to be massively beneficial for us now that we're a team of 7 engineers (as of January 2022).

- We continue to leverage GCP's products and services more and more as we grow
- GCP Cloud SQL gives us peace of mind - fully-managed DB services are fantastic
- Having staging & production environments from day 1 has allowed us to identify issues long before our users encounter them, as well as to allow us to develop in a stress-free environment
- CI/CD on CircleCI has been the key to allowing us to move quickly and have a very fast implement-test-ship feedback loop


## Adding Tools For Improved DX & TypeSafety

**Adding Prisma For A Supercharged DB Experience**

The codebase that I inherited was written in JS [with many questionable decisions](https://gist.github.com/supermacro/9cd3df429756b60208ea872f42b54734) baked into the architecture and codebase. One such decision was to not use an ORM, but instead perform raw SQL queries without even considering SQL injection attacks.

As such, the decision was made to transition away from raw SQL queries and begin using a typesafe query tool. That tool ended up being [Prisma](https://www.prisma.io/). I knew that we'd be moving over to TypeScript, and as such it just made sense to use Prisma since it generates fantastic types for you based on [Prisma's own db modelling DSL](https://www.prisma.io/docs/guides/database/prototyping-schema-db-push#prototyping-a-new-schema).

**Transitioning To TypeScript**

Over the coming weeks I would also begin converting all JS files into TypeScript files. The simple act of turning on typechecking on a file proved hugely valuable given that almost each time a file was changed from a `.js` extension to a `.ts` extension, the compiler would find errors that would have otherwise been runtime exceptions.

**TypeSafe HTTP Endpoints**

I also brought over a pattern that I use to develop typesafe HTTP endpoints. That pattern allows for a really clean and easy to follow way of implementing new routes for ExpressJS Applications. Here is an example route implemented using this pattern:


```typescript
// Docs: docs/endpoints/create-firm/README.md

import { route, AppData } from 'router'
import { createFirm } from 'resources/firm'
import { RbacPermission } from 'router/rbac'
import { createFirmParser, RawFirm } from './utils'

// Using template literal types to generate a union of
// all possible RBAC permissions
// https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html
const requiredPermissions: RbacPermission = 'create:firm'

const config = {
  // helps deserialize data from `unknown` type
  // to some type `T` that gets assigned to the `body` value below
  parser: createFirmParser,  

  // enforce RBAC, if user performing the request doesn't have
  // the required permission, our server automatically rejects
  // the request, and the request handler is not called
  requiredPermissions,
}


// `null` type argument specifies the response body
// `RawFirm` represents the incoming request body (must conform with type
//  of `parser` above)
//      ... this isn't just some flimsy typecast
export default route<null, RawFirm>(config, ({ body, utils }) =>
  createFirm(body) // returns `QueryResult<Firm>`
    .map(() => AppData.init(null)) // QueryResult is a functor that you can map over
    .mapErr(utils.intoRouteError), // can also map over the error type, which is a `DbError`

    // thus now we have a `Result<AppData<null>, RouteError>`
)
```

This `route` function implements part of the ["clean architecture" pattern](https://danuker.go.ro/the-grand-unified-theory-of-software-architecture.html) of hiding IO from both the outer layers of the system as well as the inner layers of the system. It allows for having 100% pure business logic that is easily testable.

This particular implementation of `route` is closed source, but you can find its successor over at my personal GitHub:

[supermacro/flecha](https://github.com/supermacro/flecha)

`flecha` is already in use by us at Caribou within our email notifications system.


**TypeSafe Error Modelling**

Over the coming weeks, I would begin using a simple but very powerful pattern for modelling errors in a typesafe way - Algebraic Data Types.

A few years ago I developed a library specifically for this purpose: 

[supermacro/neverthrow](https://github.com/supermacro/neverthrow)

`neverthrow` is an integral part of our tech stack and how we write code at Caribou. It's safe to say that Algebraic Data Types are here to stay at Caribou.

The `Result` type is intentionally used accross all layers of our tech stack to significantly reduce runtime exceptions and force our development team to handle errors - or at the very least acknowledge that almost anything that a developer implements has failure cases - even those computations that are not asynchronous.

Some examples for `Result` types in our system are: 

```typescript
type QueryResult<T> = ResultAsync<T, DbError>

type RouteResult<T> = ResultAsync<AppData<T>, RouteError>
```


## Team Launches Client HealthPlanner

After having launched **HA Dash** to help us do case management, our focus shifted over to executing on the vision of providing clients of financial advisors with an exceptional experience on their journey to improve their healthcare finance situation. That product was eventually given the name **HealthPlanner**.

In reality, **HealthPlanner** is an all-encompassing term that represents several applications (namely, one portal for the financial advisor, and another portal for the clients of the financial advisors), and different services.

The first order of business was to get clients to stop using email to communicate with us. Thus, the first feature in the **HealthPlanner** application would be a secure alternative to email that complied with HIPAA regulations.

In order to move quickly, we made the decision to not re-invent the wheel and instead leverage a 3rd-party tool to manage the infrastructure of messaging; [Stream](https://getstream.io/). With Stream, we were free to simply concern ourselves over the UI/UX of this secure messaging tool (which we now call **Inbox**), and not have to worry about the intricate details of modelling a communication / chat system.


![inbox](/static/images/ha-dash-inbox.png)

*This is what Inbox looks like for our staff. We modelled the UI/UX after GMail. It's virtually the same as GMail from the point of view of the end user. Yet we don't rely on SMTP. All communication is handled by Stream.*

---

Conceptually we would have two apps, one used by our internal team, and another used by the Clients of Financial Advisors.

```
          ┌────────────┐
          │            │
          │            │
          │  HA Dash   │
          │            │
          │ ┌───────┐  │
          │ │ Inbox │  │
          └─┴┬──────┴──┘
             │     ▲
             │     │
             ▼     │
          ┌─┬──────┴┬──┐
          │ │ Inbox │  │
          │ └───────┘  │
          │            │
          │  Client    │
          │ HealthPlaner
          │            │
          └────────────┘
```

This brought about an interesting question: *How do we share code between various applications*. After looking at various solutions, the decision was made to use a monorepo approach whereby various apps would be able to import from a shared `common` folder. 

Thus all `Inbox` functionality was placed inside of this `common` folder to allow for both **HA Dash** and **Client HealthPlanner** to have shared Inbox logic.

Client HealthPlanner launched around April of 2021 and has almost entirely replaced email as the communication channel between Caribou and the Clients of Financial Advisors.

2022 has **a lot** of very exciting features in the pipeline that will make the Client HealthPlanner experience even more valuable for Clients.


## Team Launches Financial Advisor HealthPlanner 

After launching a secure email alternative and introducing it to clients, we transitioned over to improving the experience for Financial Advisors and ensuring that they too are also communicating in a HIPAA-compliant way with us. 

As such, we launched account management within **HA Dash** to allow our team to:

* Create firms on a need-to basis
* Add financial advisors to those firms
* Link client cases to those firms
* Send invites to financial advisors to join their own HealthPlanner application

Thus, around July we shipped FA HealthPlanner, or as we like to call it `FHP`.

```
  ┌────────────┬────────────────────┐
  │            │                    │
  │            │◄─────────┐         │
  │  HA Dash   │          │         │
  │            │       ┌──┴─────────▼─────┐
  │ ┌───────┐  │       │ FA HealthPlanner │
  │ │ Inbox │  │       │                  │
  └─┴┬──────┴──┘       │   ┌────────┐     │
     │     ▲           │   │ Inbox  │     │
     │     │           │   └────────┘     │
     │     │           │                  │
     ▼     │           └───────────┬──────┘
  ┌─┬──────┴┬──┐          ▲        │
  │ │ Inbox │  │          │        │
  │ └───────┘  ├──────────┘        │
  │            │                   │
  │  Client    │                   │
  │ HealthPlaner ◄─────────────────┘
  │            │
  └────────────┘
```


### Team Launches Robust Email Notification System

In order to promote usage of these applications and create a habit of using our apps, we developed an email notifcation system as well. 

Developing this email notifcation system was a very fun experience because we went through a formal RFC process. I authored the Design Document while my colleague [Leza](https://github.com/lemol) combed through the proposal and provided lots of feedback and guidance along the way in order to ensure that the proposed solution addressed all of the business requirements in an effective manner.


![notifications RFC Table Of Contents](/static/images/notifs-toc.png)

*Screenshot of the Table of Contents of our most lengthy RFC / Design Doc to date*



## Other Notable Features

**Feature Flags**

Leza implemented a simple typesafe feature flagging system for the front end that has allowed us to ship large features in pieces or to beta test with a subset of our users.

This library is used extensively by our team and is quite elegant in the way that it works, however it does have its limitations. 

The RFC for this functionality is publicly available [here](https://caribouwealth.notion.site/Feature-Flags-System-37633971984c4c8e92523c54023e124c).


**Many Improvements To Inbox UI/UX Over Time**

Our colleague Daniel has submitted many improvements to make the UX of Inbox even better for all of our users. Even small things like a clean "empty state" UI when someone's Inbox is empty.


![empty inbox](/static/images/empty-inbox-ui.png)


**UX improvements for HA Dash**

Our colleague Tam implemented several UX improvements on HA Dash to make our internal staff more productive in their workflows.

--- 

## Team Grew From 1 To 4

By December, our engineering team was at 4 engineers, all distributed in various parts of the world.

* Tam in France
* Leza in Angola
* Daniel in South Africa

... And now, we're already at 7 as of January 2022!!

This is a fantastic team of high-ownership folks who love what they do and are fantastic at it. It really makes "coming into work" every day a joy.


# Setting Our Sights On Huge Goals For 2022

2022 has already kicked off with a bang. Tam rearchitected our `frontends` monorepo to allow for an even better CI/CD experience and even tighter developer feedbackloops. And we're on the verge of shipping two massive features:

* In house surveys for our clients (no longer constrained by the third-party survey tool we were using before)
* Household management for Financial Advisors (they will no longer depend on us to do certain time consumering tasks on their behalf, in addition to providing a much better user experience to our advisors)

And this is just the start ... We cannot wait to see what we accomplish this year!


# Did you enjoy this post? We're hiring!

We have several [open roles](https://caribouwealth.notion.site/Careers-at-Caribou-357783b6007f4d15bae8c7b9651cebf5) accros Ops, Design, Marketing and Engineering!

