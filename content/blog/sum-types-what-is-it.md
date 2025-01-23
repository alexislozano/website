+++
title = "Sum types, what is it?"
date = 2020-11-20
+++

And that is a good question! Let's explore what is a sum type and why there are so interesting. Note that we will use the language Elm in this article. Worry not, I'll explain everything.

## Types ?

Okay, let's begin with a simple definition of what a type is. The type of a value is its kind. So for instance, the value `3` is a number and the value `hello` is a character string. Depending on the language, the types can have different names. In Elm, here are the basic types:

```
3: Int
3.4: Float
"hello": String
True: Bool
```

## And in the darkness bind them

Almost every language also have a way to make type aggregates, or product types. Structs, classes, type aliases, all those are product types. For example in Elm, you could make type aliases for different animals:

```
type alias Cat = {
    name: String,
    age: Int
}

type alias Dog = {
    name: String,
    age: Int,
    breed: String,
}
```

Ok, that's great. But actually no. In `Dog`, the attribute `breed` has the type `String`. So, we could make a dog with the `saucisse` breed and it would be ok. To solve this problem we will use... sum types!

```
type alias Dog = {
    name: String,
    age: Int,
    breed: Breed,
}

type Breed 
    = Bulldog
    | GoldenRetriever
    | Chug
```

Nice right? The best here is that we will be able to match the breed to make different things in a function:

```
breedName: Breed -> String
breedName breed = 
    case breed of
        Bulldog -> 
            "bulldog"
        
        GoldenRetriever -> 
            "golden retriever"
        
        Chug -> 
            "chug"
```

And what is great here is that we don't need a default or else case because we know that `Breed` only contains the three breeds.

## There is more ?

Yes! What if we could put data in a sum type ? After all, we can do it with product types. Good news, that is possible. Let us take our prior example. What if we wanted to have a function which takes an animal, either dog or cat and do thing with it? We can do it:

```
type Animal 
    = Cat Cat
    | Dog Dog
```

So now we have a type `Animal` which can be of type `Cat` or `Dog`, and which contains the data of the animal. Therefore we are able to write the following function:

```
introduce: Animal -> String
introduce animal =
    case animal of
        Cat cat ->
            "Meow, I am a cat named " ++ cat.name ++ "."
        
        Dog dog ->
            "Woof, I am a " ++ breedName dog.breed ++ " named " ++ dog.name ++ "."
```

Still no default case, as we took in account every possibility.

## Conclusion

To summarize, we could say that sum types bring what we call in other languages `enum` and `abstract class` or `interface`. The big difference with the first is that we can store data of every type in a sum type. The difference with the second ones is that it is the parent which knows its children and not the other way around, therefore we don't need a default case.