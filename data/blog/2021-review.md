---
title: 2021 Year Review
date: 2022-01-08
tags: [management]
draft: true
summary: Documenting 2021's highlights of Caribou's Engineering team
---

2021 proved to be a massive success for Caribou as a whole. We managed to:

- Form a fast-paced and experienced founding engineering team with an obsession over code quality + correctness, functional programming and moving hella fast ⚡
- Ship 3 separate web applications to end users, develop an email notifications system, a feature flagging system, a secure HIPAA-complient email alternative, and much more
- Close 7 Firms, each with somewhere between \$300 million to \$2.7B in Assets Under Management!!!
- Closed our seed round of $2.5M by some of the best angels in the startup scene (round led by the Altmans)

In this post we are going to cover some of our proudest technical moments.

## First order of business: Getting Our First App In Production

After quite a lot of back-and-forth, a search for a founding engineer had come to its conclusion in January, and [Giorgio](https://www.linkedin.com/in/giorgiodelgado/) joined with a burning desire to execute on the company's vision of *creating a world where healthcare doesn't get in the way of living*, in addition to his own passion for creating high-performance & exceptional engineering teams.

At the time of joining, Caribou had a single web application that had been developed by a contractor. 

![ocean](/static/images/early-ha-dash.png)
> Screenshot of one of the pages of "HA Dash", which we use to manage cases (anything from insurance cost analysis to urgent cancer healthcare navigation and planning).

This one application became known as "HA Dash", and was being developed at the time in a waterfall style. It was deemed to be not yet "fully complete", and as such, had not yet been put to use by our internal team.



- Moving over to gcp, setting up staging and prod environments, CircleCI


Becoming more agile. Deleting code for things that weren't in use but getting in the way.






## Looking at 2022

Team's already at 7!


After a somewhat lengthy back-and-forth,  agreed to join as founding engineer.

## Founding Engineer Joins At Start Of Year


In January, Caribou's cofounders, [Christine](https://www.linkedin.com/in/christinesimone3/) and [Cory](https://www.linkedin.com/in/coryblumenfeld/) began reaching out to experienced eng folks in order to find an experienced engineer to help develop a solid engineering organization. At the time, the Caribou team was already working with various financial advisors in the US, but without any in-house software. Rather; the team was leveraging no-code tools, substantial Google Sheets and manual processes in order to deliver value to financial advisors and their clients.

 

The company had 

In February, [Giorgio]() joined as founding engineer to help develop a structured

Our eng team grew from 1 engineer to 4 (as of this writing, we're now at 7 and counting!). 




Outline

- January
    - Getting reached out to by Cory & Christine
    - Deciding to join

- Joining on February

- monorepo setup to share code between files
    - You don't need any 3rd party tools to have a monorepo setup with shared code
        - Just configure the tsc compiler to reference various top-level folders

- Getting a grasp on things and focusing on actually getting our systems to be put to use
    - Moving over to GCP
        - Inheriting a mess of a codebase
            - DB was confured using Sweedish Collation
    - Moving from Jira to Linear
    - Setting up TypeScript accross all projects
    - Introducing `neverthrow`
    - Using an improved version of parlez-vous' route function

- Goal #1 making HA Dash useable for our team
    - Actually useful case management
      - Notes & time tracking
    - Assigning folks to cases

- Shipping V1 of messaging

- Getting folks to use Client HealthPlanner

- Developing processes focused on speed and reliability

- Leza joining (July)

- Leza implementing a feature flag system
    - Link to page

- Developing notifications system (July, with Lez)
    - Leveraging an RFC / Design Document Process
    - Using `flecha` 

- Daniel joining (October)

- Tam joining  (November)
    - Tam implementing an improvement over monorepos

