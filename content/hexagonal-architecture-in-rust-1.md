+++
title = "Hexagonal architecture in Rust #1 - Domain"
date = 2021-08-21
+++

This article is part of the following series:

- Hexagonal architecture in Rust #1 - Domain
- [Hexagonal architecture in Rust #2 - In-memory repository](@/hexagonal-architecture-in-rust-2.md)
- [Hexagonal architecture in Rust #3 - HTTP API](@/hexagonal-architecture-in-rust-3.md)
- [Hexagonal architecture in Rust #4 - Refactoring](@/hexagonal-architecture-in-rust-4.md)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](@/hexagonal-architecture-in-rust-5.md)
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](@/hexagonal-architecture-in-rust-7.md)

For some time, I've been reading a lot of articles and books about hexagonal architecture, clean architecture, and so on. I've watched a lot of talks too. During all this time learning about these topics, I was wondering how I would implement them in Rust, knowing that the ownership model would maybe make it hard.

This article will probably be the first of a series where I'll show how to implement a piece of software using the patterns I've mentionned.

## Hexagonal architecture

Hexagonal architecture, onion architecture, clean architecture... are quite the same thing, so I'll refer to hexagonal architecture from now on. 

The idea is to make the core of your application independent from the dependencies. The core is often called the domain, it is where all the business rules and entities of your application are found. The dependencies are basically the rest of your application: databases, frameworks, libraries, message queues, you name it. In essence, this architecture is a way to separate the business part of your application to the implementation details.

There are several advantages to this architecture:
- you can change the domain without changing the dependencies
- you can change the dependencies without changing the domain
- you can easily test the domain
- you can think about which technical dependencies to use when the need comes, and not at the very start of your implementation

## A wild business need appears!

A morning, our client comes to us, and we begin the following conversation:

- Hi, I need a software system to manage Pokemons.
- Okay, what do you want to do with these Pokemons?
- I need to create new Pokemons, delete them, and retrieve them.
- Noted. How would you want to access your system? Using a web interface, using the terminal?
- Erm, I don't really know...
- And where do you want to store the Pokemons? Do you have a database or an account on an object storage service?
- What's a database?

Here, you could say the client does not know what he wants. But the thing is, for now, we don't really need the answer to these technical questions. The important things are the use cases. Let us rewrite them:

- Create a Pokemon
- Read all Pokemons
- Read one Pokemon
- Delete a Pokemon

## Our first use case

Our project will be implemented in Rust, hence the title ;) Let's first create a new binary project:

```
cargo new pokedex
```

And then let's create our first use case:

```
src
├── domain
│   ├── create_pokemon.rs
│   └── mod.rs
└── main.rs
```

And don't forget the modules:

```
// main.rs
mod domain;

// domain/mod.rs
mod create_pokemon;
```

What I like to do is writing first the tests as if the code was already written. It helps me to create a clean API. So we can open `domain/create_pokemon.rs` and add our first test:

```
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_should_return_the_pokemon_number_otherwise() {
        let number = 25;
        let req = Request {
            number,
            name: String::from("Pikachu"),
            types: vec![String::from("Electric")],
        };

        let res = execute(req);

        assert_eq!(res, number);
    }
}
```

Of course, it does not compile here. First thing is we need to create a `Request` struct:

```
struct Request {
    number: u16,
    name: String,
    types: Vec<String>,
}
```

Note that we don't use fancy types in the `Request` struct. Why? Because we don't want the code which call our use cases to know about the domain entites. As I've written before, the goal is to have an independent domain layer. 

Now, we need to implement the `execute` function:

```
fn execute(req: Request) -> u16 {
    req.number
}
```

It works! Let's give that to our client! I'm not sure he will be happy to get this result. Actually, we never check that the `Request` is good. What if the number is not in the correct bounds? What if the given name is an empty string? What if one of the types do not exist in the Pokemon world? Let's fix that :)

## Entities

Let's add a new test to check the use case will return an error when given a bad formed `Request`:

```
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    let req = Request {
        number: 25,
        name: String::from(""),
        types: vec![String::from("Electric")],
    };

    let res = execute(req);

    match res {
        Response::BadRequest => {}
        _ => unreachable!(),
    };
}
```

It does not compile because `Response` does not exist yet. For now the use case is only returning a `u16` so we`ll have to change that:

```
enum Response {
    Ok(u16),
    BadRequest,
}

fn execute(req: Request) -> Response {
    Response::BadRequest
}
```

We should also change the other test to make it check the `Ok` case:

```
match res {
    Response::Ok(res_number) => assert_eq!(res_number, number),
    _ => unreachable!(),
};
```

Now, the code compiles! But the test checking the `Ok` case fails. That's logical: we always return `Response::BadRequest`. We will come back later to this test. For now, we will define the business rules of the values we get in the `Request`. Let's create a new file `domain/entities.rs` where we'll store them.

```
└── domain
    ├── create_pokemon.rs
    ├── entities.rs
    └── mod.rs
```

### Pokemon number

The number has to be `> 0` and `< 899`:

```
pub struct PokemonNumber(u16);

impl TryFrom<u16> for PokemonNumber {
    type Error = ();

    fn try_from(n: u16) -> Result<Self, Self::Error> {
        if n > 0 && n < 899 {
            Ok(Self(n))
        } else {
            Err(())
        }
    }
}

impl From<PokemonNumber> for u16 {
    fn from(n: PokemonNumber) -> u16 {
        n.0
    }
}
```

### Pokemon name

The name cannot be an empty string:

```
pub struct PokemonName(String);

impl TryFrom<String> for PokemonName {
    type Error = ();

    fn try_from(n: String) -> Result<Self, Self::Error> {
        if n.is_empty() {
            Err(())
        } else {
            Ok(Self(n))
        }
    }
}
```

### Pokemon types

The types cannot be an empty list and each type should be one of the defined Pokemon types. For now we'll only define the `Electric` type:

```
pub struct PokemonTypes(Vec<PokemonType>);

impl TryFrom<Vec<String>> for PokemonTypes {
    type Error = ();

    fn try_from(ts: Vec<String>) -> Result<Self, Self::Error> {
        if ts.is_empty() {
            Err(())
        } else {
            let mut pts = vec![];
            for t in ts.iter() {
                match PokemonType::try_from(String::from(t)) {
                    Ok(pt) => pts.push(pt),
                    _ => return Err(()),
                }
            }
            Ok(Self(pts))
        }
    }
}

enum PokemonType {
    Electric,
}

impl TryFrom<String> for PokemonType {
    type Error = ();

    fn try_from(t: String) -> Result<Self, Self::Error> {
        match t.as_str() {
            "Electric" => Ok(Self::Electric),
            _ => Err(()),
        }
    }
}
```

We can now use our entities in the use case: 

```
fn execute(req: Request) -> Response {
    match (
        PokemonNumber::try_from(req.number),
        PokemonName::try_from(req.name),
        PokemonTypes::try_from(req.types),
    ) {
        (Ok(number), Ok(_), Ok(_)) => Response::Ok(u16::from(number)),
        _ => Response::BadRequest,
    }
}
```

And all the tests now pass!

## Next steps

In the next articles, we'll see how to implement multiple repositories to store pokemons. All the repositories will implement the same trait and thus they will be easily pluggable and exchangeable. We'll also implement multiple front-ends to the use cases to be able to work with Pokemons from different interfaces.

The code we've written here is accessible on [github](https://github.com/alexislozano/pokedex/tree/article-1).