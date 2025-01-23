+++
title = "Sum types, but this time in TypeScript"
date = 2020-12-29
+++

This will just be a wall of code, nothing more. If you have read my article about [sum types](@/blog/sum-types-what-is-it.md), you know what sum types are. Following is just my attempt to the implementation in TypeScript of the Elm code I have previously written.

```
type Animal 
    = Dog 
    | Cat

enum AnimalType {
    DOG,
    CAT
}
    
type Dog = {
    type: AnimalType.DOG,
    name: string,
    age: number,
    breed: Breed,
}

enum Breed {
    BULLDOG,
    GOLDEN_RETRIEVER,
    CHUG
}

type Cat = {
    type: AnimalType.CAT,
    name: string,
    age: number,
}

function breedName(breed: Breed): string {
    switch(breed) {
        case(Breed.BULLDOG): return "bulldog";
        case(Breed.GOLDEN_RETRIEVER): return "golden retriever";
        case(Breed.CHUG): return "chug";
    }
}

function introduce(animal: Animal): string {
    switch (animal.type) {
        case(AnimalType.CAT): return `Meow, I am a cat named ${animal.name}.`;
        case(AnimalType.DOG): return `Woof, I am a ${breedName(animal.breed)} named ${animal.name}.`;
    }
}

console.log(introduce({ type: AnimalType.CAT, name: "Jean-Pierre", age: 12 }));
console.log(introduce({ type: AnimalType.DOG, name: "Philippe", breed: "GoldenRetriever" ,age: 9 }));
```

It is pretty nice and looks like the Elm code. The only real difference I can see, taking aside the syntax, is that I needed to add a `type` attribute in my `Dog` and `Cat` types because TypeScript does not support matching on the type directly (see the `introduce` function).
