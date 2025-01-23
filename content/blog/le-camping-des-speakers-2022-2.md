+++
title = "Le camping des speakers - Day 2"
date = 2022-06-10
+++

This post is in two parts:
- [Le camping des speakers - Day 1](@/blog/le-camping-des-speakers-2022-1.md)
- Le camping des speakers - Day 2

In this article, I'm going to write my notes about this second and last day and the conferences I have attended.

## La potion magique pour faire progresser ta carrière (The magic potion to advance your career) - [David Pilato](https://twitter.com/dadoonet)

A great presentation of the life of David, where he showed what helped him to be where he is now:
- Be visible
- Contribute
- Show and attract love from your communities

## Svelte.js : Le compilateur en guise de Framework (Svelte.js: compiler as a framework) - [Thibault Goudouneix](https://twitter.com/nilmanduil)

The first version of svelte has been released very fast after the first few commits.

Pros:
- compiler
- no shadow dom
- no runtime

So it is really fast compared to react and others. A bit like Elm maybe ;)

Cons:
- the community is still small

He showed us how the code was written, it really reminded vue.js: one-file components in three parts for the script, the style, and the template. A store is integrated in svelte.js.

Future:
- accessibility
- internationalization
- cli
- svelte native (mobile)
- webgl
- SvelteKit (Next counterpart)

## Mission impossible : créer son pipeline CI/CD en quelques minutes (Impossible mission: create a CI/CD pipelines in a few minutes) - [Thomas Boni](https://twitter.com/thomas_boni)

The big issue for teams of developers is the deployment. For instance, it is pretty hard to create a deployment pipeline for gitlab.

Thomas introduced r2devops, a kind of docker hub for pipeline jobs. Using the `include` command in CI files, it is possible to import pre defined jobs in a pipeline. A good idea in this product is that all the variables can be overriden to make it bend to what the user needs.

The main take away from this talk is that CI jobs should be centralized, documented, and versioned in a company to make them reusable and easy to edit.

## Développer une culture Tech engagée - retour sur lbc², première conférence tech par leboncoin (Developping a committed tech culture - feedback on lbc², first tech conference by leboncoin) - [Guillaume Grillat](https://twitter.com/grillatg)

The talk was very interesting with a lot of ideas. Here's what I got, in non particular order:
- find the teams and their pain points
- find people who want to speak up
- talk about positive but also negative outcomes
- company should publicly communicate about their values and engagements: for instance, in a company where LGBT+ communities are not endorsed, the individuals may hide their identity, adding a mental weight on themselves
- create support groups

## Conclusion

This first edition of Le camping des speakers was great, and I hope we'll have new editions in the next years :D
