+++
title = "Very Tech Trip 2023"
date = 2023-03-18
+++

Minutes gathered from attending and speaking at [Very Tech Trip](https://verytechtrip.com/), a tech conference at Paris, France, on February the 2nd of 2023. The talks I've attended were given in French, so some resources will be in French.

## Data, AI... what if they were the solution to understand sign language? - [Eléa Petton](https://twitter.com/EleaPetton) & [Claire Gallot](https://www.linkedin.com/in/claire-gallot-b80b72153/)

In this talk, Eléa and Claire show how to use AI to recognize letters from the American Sign Language.

- Recording: not released

Some notes:

- [The American Sign Language dataset v1](https://public.roboflow.com/object-detection/american-sign-language-letters/1) contains 1728 images of letters signed with hands
- To detect objects, there are multiple methods
	- Classification and localization
	- Object detection
	- Instance segmentation
- YOLO (You Only Look Once) is a real time algorithm which is often used
- They've tried to train it with the dataset: the result were not bad but they can be improved
	- misidentification of letters
	- low precision
	- false object recognition
- Why?
	- The dataset is too small
	- The picture are not diverse enough
- The solution: data augmentation
	- horizontal flip
	- shift
	- change brightness, contrast and saturation
	- add blur and noise
- The image generation was too slow so they've used SPARK to parallelize the work

## Developing a website in public: some feedback - Myself 👋

I have talked about a side project and an epic battle against bots.

- [Recording](https://www.youtube.com/watch?v=iI_yufs4yYA)

## Security incidents: what about the human? - [Sébastien Mériot](https://twitter.com/smeriot)

Sébastien discusses about the humans dealing with security incidents.

- Recording: not released

Some notes:

- Crisis organization
	- Security incident response team
	- Infra experts
	- Crisis communication experts
	- DPO
	- People able to take the difficult decisions
- Main challenges
	- 24/7 org
	- work under pressure
	- labor laws
- Ability to deal with security incidents: people don't behave the same way when facing an incident
- Blaming is counterproductive: don't look back in anger
- Work under pressure: the situation needs us to be fully involved and get back to work quickly to run the marathon
- Post-incident: time to release the pressure and think about what happened. Decompression can lead to a feeling of unease
- Burnout in cybersecurity (France):
	- 51% experience extreme stress or burnout in 2021
	- 65% thinks about leaving this job because of job stress
	- 67% wouldn't recommend a career in this industry
- Causes
	- mentally exhausted
	- trauma
	- stress
	- post incident action plan

## From the first line of code to success: feedback about an open source project - [Flavien Normand](https://twitter.com/Supamiu_)

Flavien leads an open source webapp which helps FFXIV players to collect and build resources.

- Recording: not released

Some notes:

- Some figures
	- created in 2017
	- 400K active users per month
	- 4M of lists (main object in the app) created
	- 10K commits
	- 10 languages
	- 75 contributors
	- 26K discord users
- Some technical considerations
	- went from material UI to ant design for the design system
	- uses electron to create real time game overlays
- Costs
	- 25K€ of cloud billing since the start of the project
- Finance
	- Patreon
	- Ads (no gifs, no animations, no adult content)
- His company gives him some days to work on the project because it is open source
- Conclusion
	- Pros
		- international community
		- personal sandbox
		- great for recruitment processes
	- Cons
		- almost no contact with the game editor
		- takes a lot of time (~6Kh)
		- lot of stress

## From social work to the tech industry: advocacy of atypical profiles - [Magali Milbergue](https://twitter.com/MagaliMilbergue)

Magali talks about the profiles we don't think about when thinking about the tech industry

- Recording: not released

Some notes:

- Anecdote: Magali is big. In a conference, she had to ask the organizers for a chair without arms. For the whole day, she had to make sure nobody was moving her chair or sitting in it. Big mental workload.
- Typical dev:
	- white
	- man
	- heterosexual
	- cisgender
	- able-bodied
	- young
	- neurotypic
	- advanced education
- Anecdote: Phone which unlock with face recognition were not recognizing people with facial tattoos or one-eyed people.
- Magali's story during her retraining to become a developer:
	- she was pushed to do frontend work whereas she preferred backend
	- she was using a pink theme on her IDE and other people were finding it weird
- Bias:
	- Definition: distortion in information processing which may have consequences on reasoning and decisions
	- Good: a oven can burn you, it's dangerous
	- Bad: I let you find your own ;)
- The tech industry is not diverse. This creates issues on products which are not tested on a diverse enough team. For instance, accessibility is not handled correctly because there are not enough disabled people in tech teams. Do we need to include atypical profiles in the tech industry: yes and it's a matter of survival for the sector.
