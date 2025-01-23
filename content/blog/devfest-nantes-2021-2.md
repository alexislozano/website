+++
title = "DevFest Nantes 2021 - Day 2"
date = 2021-10-22
+++

This post is in two parts:
- [DevFest Nantes 2021 - Day 1](@/blog/devfest-nantes-2021-1.md)
- DevFest Nantes 2021 - Day 2

In this post, I'll write the notes I took during the conferences I attended to. Have a nice read :)

## Breakfast

Much croissant, such gâteaux.

## Coupable de code legacy en JS: Comment s'en sortir? (Culprit of legacy code in JS: How to get through it?) - [Adrien Joly](https://twitter.com/adrienjoly)

Some definitions to begin with:
- legacy code: valuable code you're afraid of
- code smell: surface indication that usually correspond to a deeper problem in the system
- technical debt: refactoring effort required to add a new feature non invasively
- refactoring: basically change the code for good without changing features

Here's the steps he got through during the presentation:
- write approval tests against the old code (did that before the presentation)
- remove unused code like commented code for instance
- extract functions
- split code doing multiple things
- replace implicits with explicits

And the pieces of advice he gave us:
- define and protect the functional perimeter
- observe, cartography and plan
- delete dead code
- facilitate the changes
- apply one refactoring method at the time

## Hasura: le GraphQL qui hassure (Hasura: GraphQL which kicks hass) - [Hugo Wood](https://twitter.com/mercury_wood) & [Valentin Cocaud](https://twitter.com/ragorn44)

The talk began with a small feature tour of GraphQL:

- avoid request cascades
- schemas
- avoid fetching whole objects

Then the speakers focused on Hasura. So what is it?

- server which connects to an SQL database and expose a GraphQL API
- open source

All the CRUD methods are present, nothing to see here.

How to implement domain code:

- permissions
- SQL functions (did not know it existed)
- delegate the action to an external backend (eg. rendering PDF, sending emails)
- send events when something changes in the database with subscriptions

How to implement authentification (Hasura uses JWT internally):
- custom backend
- 3rd party API (Auth0)
- SQL functions (you need to be a bit weird to do that :p)

Versionning: Hasura uses config files which can be versionned. Also, the changes you make in the Hasura interface are logged in files so they can be versionned too.

Integration with existing IT:
- functions to translate REST calls to GraphQL calls directly in Hasura
- multi database aggregation (with cross joins!)

Performance:
- all you do with Hasura is compiled to SQL requests so no bloat
- be careful about subscriptions which are not cached

## Notre recette de l'equipe parfaite (Our recipe for a perfect team) - [Estelle Landry](https://twitter.com/estelandry) & [Yvonnick Frin](https://twitter.com/YvonnickFrin)

The speakers made a lot of analogies with cooking. It was a bit weird but hey, they looked like having a lot of fun :) Estelle and Yvonnick work at Pix, they wanted to share their values: quality, agility, user-centricity, team spirit. The talk was about how they handled their work at Pix.

The first part of the work is the discovery:
- create a roadmap with sprints and epics (align the team on the vision)
- transform user needs to features to develop (alignment with all the actors)

The second part is development:
- split the epics in atomic user stories following the Pareto principle (easier reviews, short leadtime, faster feedback)
- prioritise tickets (quick wins, user feedback, technical debt...)
- use pair and mod programming

The third part is deployment:
- the PO does it with a one button action on Slack
- staging and prod environments (one deployment for 8 to 10 tickets = 1 to 2 days)

## Lunch

The deserts were awesome!

## The blind test - Florent Lévêque & Hervé Boisgontier

Florent is a blind person and a web development student. Hervé and him talked about the need to make the web accessible and showcased how screen readers were used and what to do to improve the experience for disabled people.

Two devices to "see" what's on screen:
- screen readers
- braille devices

How to help people browsing the web:
- semantic HTML
- labels on fields
- fieldset and legends on forms
- aria-label on links and buttons
- aria-describeby and role alert for fields erroring out

## La dette UI: en finir avec les interfaces jetables et concevoir une UI durable, adaptable et structurée (UI debt: get out of the throw away interfaces and design a lasting, adjustable and structured UI) - [Loïc Vanderschooten](https://twitter.com/CaptainFloax)

Example of design debt symptoms:
- elements which do the same things but with different styles
- elements which need some expert to make changes (illustrations)
- elements positionned in a precise context

2 components to the UI debt:
- artistic aspect which is hard to frame
- field not used to product structuration

The UI, UX, and technical debts all go together.

What is often seen:
- art centered process, no fixed boundaries
- no vison on a long term interface
- no structure / nomenclature in the files
- too short timeframes = no further anticipation
- no measurement of UI debt to have actionable figures

The smells:
- too many reference files
- no docs or processes for evolution
- too many actors needed to make changes
- non editable products
- complex settings and editing
- easier to begin from scratch than to edit the existing design system

Techniques:
- appoint a designOps master
- know the impact a change could have
- anticipate

How to begin:
- check the existing design system
- choose what'll be kept or not
- appoint the designOps master
- define a design framework
- write clear processes
- write an evolution plan
- name layers
- versionning
- work in iterations

Errors to avoid:
- always change for new tools
- use outdated or personal processes
- do not adapt actions to context
- try to remove all debt
- do not talk to the other teams

## J'ai presque fini ! (I'm almost done!) - [David Laizé](https://twitter.com/DavidLaize)

How to work with a colleague who always says "I'm almost done" when obviously he did says that for two weeks.

Two cognitive bias:

- overconfidence effect
  - to help: what has changed compared to yesterday / would you make a bet on that
  - pair programming  
- escalation of commitment
  - mentoring

It is really hard to notice these bias on ourselves but it is easy to find them in other people. The solution is benevolent communication.

## Trucs et astuces pour réussir son burn-out (Tips and tricks for a successful burn-out) - [Cynthia Staebler](https://twitter.com/CynthiaStaebler) & [Julia Lehoux](https://twitter.com/julia_lehoux)

What are the causes of burn-out:
- personnality
- inter personnal relationships
- professional environment
- society

490 000 burn-outs / year in France.

What to do:
- choose bad relationships
- loose your self trust
- get into a dominating / dominated scenario
- get harassed

More things to do:
- no boundaries
- say yes even when you don't want to
- do not disconnect, ever
- take everything personnaly

That's the recipe for a perfect burn-out I'll sure you love <3

## Ending keynote

Fours speakers were on the scene. They were asked questions. The three loosers got great books about CakePHP, Flash and Access :'D The winner got a CommitStrip book.