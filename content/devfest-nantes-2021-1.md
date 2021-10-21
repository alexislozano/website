+++
title = "DevFest Nantes 2021 - Day 1"
date = 2021-10-21
+++

Today was the first day of the 2021 edition of the DevFest in Nantes, France. There were a lot of conferences and attendees. And I was there, in the public, taking notes about what I was learning.

So this post is just that: my notes. Hope you'll like it.

## Breakfast

Nice croissants :)

## Opening Keynote - [Antonin Fourneau](https://twitter.com/AntoninFourneau)

Antonin Fourneau is a "bricodeur" as he calls himself. In English, I would translate it to "makoder". He likes to mix technology and art in his work. He showcased several pieces of art he worked on:

- a makey-makey like device
- a net of optical fiber enclosed into concrete showing the name of people who died in war
- a led wall which reacts to water
- a led wall which reacts to sound

He showed his love for video games by quoting his spiritual father Gunpei Yokoi. 

## Component Driven Development - [Debbie O'Brien](https://twitter.com/debs_obrien)

The premise of this talk is: building web apps in 2021 is still a mess.

We have evolved from a monolith for the whole application, to a 2-tier model: one part is the frontend and the other one is the backend. In the backend, it then evolved to micro-services: small and reusable components. However, in the front-end, we still have monoliths.

The front-end repository is a monolith which uses components. Components are great as they are reusable. The thing is they are nested is the front-end repository. This causes the following issues:

- one codebase = no team ownership
- one codebase = one version
- one codebase = one build

How would we solve these issues? For example in the context of reusing code in several apps:

- let's just duplicate the components -> coherence problems
- let's package components and deploy them on npm -> bad discoverability for developers
- let's use a monorepo
    - pro: the code is shared between modules
    - con: slow builds, not clear code ownership

The solution to that is first a team solution: let's split the team into feature teams. Each team would have its own repository. That implies:

- separate CI pipelines
- indidually shipped components
- independantly versionned components

To help with that, Debbie then talked a lot about the company she's working for: Bit. And to me the Bit part was a bit (pun intended :p) too long.

## Level Unlocked: GitOps to the Edge and Infrastructure Provisioning - [Katie Gamanji](https://twitter.com/k_gamanji)

During this talk, Katie talked about CNCF tools and how they could help the deployment of infrastructures using GitOps.

The mantra of the conference was: the git repo is the source of truth of the state of the application / infrastructure.

Here's what I could note about some of the tools she talked about:

- ClusterAPI: declarative management of Kubernetes clusters
- KubeEdge: synchronisation of state between the cloud and the edges

I had a question in the end: what is the difference between these tools and something like Terraform or Ansible. Her answer was that the tools she talked about was part of the Cloud Native environment and so they were specifically designed for Kubernetes and cloud providers.

## Lunch

Good crumble, great macarons, yum yum :3

## Don't miss the Deno Train - [M4DZ](https://twitter.com/m4d_z)

This conference was a bit complex as there were a lot of things to say. So maybe I did not get everything right, tell me on Twitter if I'm badly mistaken :p

So first slide begins with: "Javascript sucks". Yep. So M4DZ talks about the problems of Javascript and node:

- monothreaded
- callback hell
- no security by default
- gyp is bad
- npm and package.json are meh
- internal black magic

So what would be a better take on this? The things we'd like to have would be:

- good I/O
- security by default
- message passing

Enters Deno:

- typescript is first-class citizen
- no more npm -> sweet imports from friendly CDNs
- opt-in flags for security

Then he talked about the ecosystem and the libraries we could use instead of the well-known node libraries. After that, he told us a lot about Aleph (kind of Next.js on Deno) and SSR. Then about web components. And I will be honest to say it was hard to follow first because I don't know well the node ecosystem, and second because digestion kicked in.

## 🦀 Rust, un choix possible ? (Rust, a possible choice?) - [Charles-Henri Guérin](https://twitter.com/charlyx) & [Pierre-Yves Aillet](https://twitter.com/pyaillet)

Yes, crabs! I will go fast on this one as it was mostly a small language feature tour:

- ownership: 
    - auto drops at the end of the variables scope
- reference rules:
    - no multiple mutable references
    - no use after free
- type safety with enums
    - pattern matching
    - exhaustive matches
- fearless concurrency
    - threads
    - channels
    - mutexes

They did some live-coding to demonstrate all the above mentionned properties (no concurrency though, not enough time for that). After that they focused on the compiler messages, the tooling, and how to promote Rust in the company we work for.

## Dans le cloud tout est managé mais pas la sécurité (In the cloud, everything is managed but the security is not) - [Éric Briand](https://twitter.com/eric_briand)

That was a very interesting talk. Éric first introduced the three pillars of InfoSec:

- availability
- integrity
- confidentiality

That reminds me of school... Kay, let's move on. Before the cloud, the infrastructure was on premise, and the security model was the one of the Fortress. Big walls (firewall) and no security inside. In the cloud, this model is not coherent anymore: there is no more physical separation between our clients or between you and your competitors.

Following are the responsibilities your cloud provider has:

- physical security of datacenters
- background check of employees
- suspect activities monitoring 
- isolation between data and clients
- should work :p

And the ones the client has:

- prevent
    - identity management
    - user rights and roles
    - security policies
    - data and flow encryption
    - employees learning
- detect
    - audit, traces
    - management implication
    - vulnerability scan
    - independent audits
- fix
    - communicate when data leaks
    - delete access from compromised and reset credentials
    - iterate on security policies

To develop with security best practices in mind: check OWASP.

Éric then talked about two stories.

### Linkedin in 2012

A hacker got the credentials of one employee. All employees had access to everything. The hacker dumped the whole credential database. 164 millions user accounts were leaked. These credentials were then used to try to connect to other services accounts.

What were the problems here:

- credential management
    - lacks least privilege policy
    - lacks multi-factor auth
    - lack captchas to ban bots
- encryption
    - data was hashed without salt


What are the solutions:

- monitoring and alerting
- isolation with containers and VPCs
- security audits

### Mexican voting bureau in 2016

A mongodb instance was totally open because misconfigured. 93 millions of citizen's data were leaked. Every data could be read, updated and deleted.

When deploying something, we need to know it to be sure the default config is not too permissive.

Other stories about Disney+, the Dow Jones, and Tesla.

In conclusion, the cloud offers more possibilities and with more powers come great reponsibility.

I had a question at the end of the talk: For a small structure, would you advise to use PaaS infrastructures to mitigate security issues? The answer was a big yes, with the emphasis on the fact that you'll still have issues when becoming bigger. Exemple of Pix getting DDOS issues.

## Go Generics - [Benoît Masson](https://twitter.com/bmasson35)

We talk about it for 10 years, and generics will at last be here in the next version!

Generics already exist with `interface{}` but it not great to say the least.

My question: Do you think we'll soon have union types? His answer: I don't know but I hope so. Generics are a first step though.

## OpenAPI & AsyncAPI : spécifications et contrats (OpenAPI & AsyncAPI: specifications and contracts) - [Sébastien Charrier](https://twitter.com/scharrier)

Read and write API docs sucks. Today, writing tests has become a part of a developer job, but writing API docs is still not there.

Often, docs are outdated, split between places, and more than often, they don't exist. This issue is not a documentation problem, it is actually a problem of human and technical synchronisation.

Nowadays, the more used spec for REST is called OpenAPI (ex Swagger). To write it, you have two choices:

- code first: genrated from the code
- design first: manually written -> Sébastien prefers this one

With the spec, you can:

- generate docs
- mock the API for testing

AsyncAPI is kind of OpenAPI but for event based APIs.

## Conclusion

This first day of DevFest Nantes was pretty long but really interesting. Tomorrow will be the second and last day :)