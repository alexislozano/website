+++
title = "Le camping des speakers - Day 1"
date = 2022-06-11
+++

Thursday was the first day of Le camping des speakers, 1st edition. It took place near Vannes, France. And I was there as an attendee, with my ears open.

In this article, I'm going to write my notes about this first day and the conferences I have attended.

## Opening keynote - [Horacio Gonzalez](https://twitter.com/LostInBrittany) & [Pierre Tibulle](https://twitter.com/ptibulle)

Some slides about the sponsors, and some explanations about what was going to happen during the two days. Then, all the Thursday's speakers gave a synopsis of their talks one-by-one, trying to get attendees to come.

## WebAssembly n'est pas qu'une affaire de frontend (WebAssembly is not frontend only) - [Benjamin Coenen](https://twitter.com/bnj25)

WebAssembly has been created at Mozilla. With the lay-off, the team is now working at Fastly.

It is a shared-stack language with basic types only, so compilers have to convert complex types (strings, structs, ADTs, etc) to ints, floats, and bytes.

There are two files formats:
- `.wasm`: binary run by the browser
- `.wat`: text representation, human-readable

To compile to `.wasm`, there are mostly four languages:
- Rust
- Go
- C
- AssemblyScript

There are some constraints we have to work with:
- no async
- no garbage collector: garbage collected languages like Go or AssemblyScript need to embed a runtime in the `.wasm` binary

Use cases for WebAssembly in the backend:
- FAAS
- smart contracts
- plugins for other languages through FFI

`.wasm` binaries are normally run sandboxed. To go around this issue in the backend and be able to access IO, we can use WASI.

## La cuisine m'a sauvé du burn-out (Cooking saved me from burn-out) - [Yannick Guern](https://twitter.com/_Akanoa_)

Imposter syndrome and savior complex do not go well together. Yannick worked too hard and tried to help everyone at work. He actually had the whole codebase of his company in his head. He eventually snapped.

Helped by cooking and watching cooking shows (Street Food, Food Wars, Chef Michel Dumas' youtube videos), he learned to cook and took pleasure from it, gradually recovering.

## Ressuscitons les ordinosaures ! (Bring back computosaurs!) - [Olivier Poncet](https://twitter.com/ponceto91)

- Simulator = the user believes he uses the original device
- Emulator = The software believes it runs on the original device

What to do first in order to create an emulator:
- get the most documentation / manuals / technical specs you can
- do some reverse engineering
- choose the destination OS
- choose the destination language
- think about the GUI technology you'll use if needed

There are several sub systems in a device:
- central: ram / rom / cpu
- video
- audio
- peripherals

The hardest part to emulate is the central sub system. More than the basics you'll also need to emulate flags (internal and external), and have extra care working with timings. Plus, some games could exploit some non documented bugs, so reverse engineering can really help.

To run the guest code, there are 4 techniques:
- interpreter: mainly a big switch, it is a bit slow compared to the other techniques. [Here's](https://github.com/alexislozano/chip8) an interpreter I've coded for chip8.
- static translation: dump the code and recompile it on your destination OS
- dynamic translation: JIT compilation using memoisation
- virtualization: QEMU does that

The video sub system synchronizes the emulator with the display frames. If needed, some frames can be omitted to increase perfs.

The audio subsystem does the same for sound. You should not omit samples, or the produced sound will be very unpleasant to the user.

The peripheral sub system is often easy to emulate. It groups keyboards, joysticks, IO systems, etc...

## J'ai plié mon smartphone et il ne s'est pas cassé... mais ma web app si (I've bent my smartphone and it did not break... but my web app did) - [Olivier Leplus](https://twitter.com/olivierleplus) and [Yohan Lasorsa](https://twitter.com/sinedied)

They began with a quick story about screen sizes and formats.

Media queries can be used in css to take into account the foldable devices. Some properties are accessible using js.

There is a posture API to check what posture is the phone in: flat / laptop / tent / book

They showed us some demos with actual devices and on virtual devices.

There are various use cases:
- list view + details
- map mode a bit like in mario kart

## App-Elles - Exemple de tech for good mobile à l'heure du confinement (App-Elles, a tech for good example in the lockdown context) - [Robin Caroff](https://twitter.com/robin.caroff)

With the lockdowns, it was really hard for social workers and people in need to meet.

His organization, Resonantes, has created a mobile app to fight family violences.

Developers can also help other organizations and did. Some examples:
- the Red Cross
- Soli
- Makair

## Lead Dev, 3 ans d'xp, et alors ? (Lead dev with 3 YoE, so what?) - [Lise Quesnel](https://twitter.com/QuesnelLise)

The definition of a lead dev can be different from a company to another. Mainly:
- tech lead = technical expertise
- lead dev = team support

Different aspects to a lead dev:
- technical reference
- helps with communication in the team
- coaching
- training
- product

A lead dev needs a lot of soft skills. There are some difficulties:
- external communication
- less coding (min 40%)
- meetings
- conflict management

## Conclusion

That's all for the first day. Of course we also had some good food (burgers and sushi) and we had access to the swimming pool \o/