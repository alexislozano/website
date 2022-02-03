+++
title = "Hexagonal architecture in Rust #2 - In-memory repository"
date = 2021-08-24
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1 - Domain](@/hexagonal-architecture-in-rust-1.md)
- Hexagonal architecture in Rust #2 - In-memory repository
- [Hexagonal architecture in Rust #3 - HTTP API](@/hexagonal-architecture-in-rust-3.md)
- [Hexagonal architecture in Rust #4 - Refactoring](@/hexagonal-architecture-in-rust-4.md)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](@/hexagonal-architecture-in-rust-5.md)
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](@/hexagonal-architecture-in-rust-7.md)

> Disclaimer: In this article, I'll use a simple mutable reference for the repository. That's because for now we are just using it in tests. I'll make the relevant changes in a next article ;)

In the previous article, I've begun with creating the basic project and architecture. We have a domain module with one use case and some entities:

```
src
├── domain
│   ├── create_pokemon.rs
│   ├── entities.rs
│   └── mod.rs
└── main.rs
```

## In-memory repository

Let's go back to our `create_pokemon` use case. For now, it can return the number of the Pokemon on success or an error when the request does not conform to the business rules. It does not really store the Pokemon anywhere though. Let's fix this! And now you should know what I like to begin with: a test :) This test will check that we cannot have two Pokemons with the same number.

```
use crate::repositories::pokemon::InMemoryRepository;

#[test]
fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
    let number = PokemonNumber::try_from(25).unwrap();
    let name = PokemonName::try_from(String::from("Pikachu")).unwrap();
    let types = PokemonTypes::try_from(vec![String::from("Electric")]).unwrap();
    let mut repo = InMemoryRepository::new();
    repo.insert(number, name, types);
    let req = Request {
        number: 25,
        name: String::from("Charmander"),
        types: vec![String::from("Fire")],
    };

    let res = execute(&mut repo, req);

    match res {
        Response::Conflict => {}
        _ => unreachable!(),
    }
}
```

Here, we add a Pokemon directly in the repository. Then we try to insert a Pokemon with the same number using the use case. The use case should return a Conflict error.

As usual, it won't compile because a lot of code here is not even existing. Let's begin by adding the `Conflict` variant to the `Response`:

```
enum Response {
    ...
    Conflict,
}
```

Let's also add the `Fire` variant to `PokemonType`:

```
enum PokemonType {
    Electric,
    Fire,
}

impl TryFrom<String> for PokemonType {
    type Error = ();

    fn try_from(t: String) -> Result<Self, Self::Error> {
        match t.as_str() {
            "Electric" => Ok(Self::Electric),
            "Fire" => Ok(Self::Fire),
            _ => Err(()),
        }
    }
}
```

You probably wonder what `InMemoryRepository` is. Remember we talked about having repositories the use cases would use without knowing their implementation. This is our first implementation that'll be used mainly for tests. It'll still work like a real repository so we will be able to use it to show our progress to the client and ask him for feedback. As you can see, the repo is sent to the use case as an argument. Let's add it:

```
use crate::repositories::pokemon::Repository;

fn execute(repo: &mut dyn Repository, req: Request) -> Response {
```

The important thing to note here is that the `execute` function does not know the implementation it gets but only its interface. Let's add the repository in the other tests:

```
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    let mut repo = InMemoryRepository::new();
    let req = Request {
    ...
    let res = execute(&mut repo, req);
    ...
}

#[test]
fn it_should_return_the_pokemon_number_otherwise() {
    let mut repo = InMemoryRepository::new();
    let number = 25;
    ...
    let res = execute(&mut repo, req);
    ...
}
```

Neither `InMemoryRepository` nor `Repository` are defined. Let's do it in a new module `repositories/pokemon.rs`:

```
src
├── domain
│   ├── create_pokemon.rs
│   ├── entities.rs
│   └── mod.rs
├── repositories
│   └── pokemon.rs
└── main.rs
```

```
pub trait Repository {}

pub struct InMemoryRepository;

impl Repository for InMemoryRepository {}
```

Let's not forget the module magic:

```
// main.rs
mod repositories;

// repositories/mod.rs
pub mod pokemon;
```

Trying running the tests, the compiler tells us the `new` function is not defined in `InMemoryRepository`, let's do it. Here, the implementation will simply store an array of Pokemons:

```
use crate::domain::entities::Pokemon;

pub struct InMemoryRepository {
    pokemons: Vec<Pokemon>,
}

impl InMemoryRepository {
    pub fn new() -> Self {
        let pokemons: Vec<Pokemon> = vec![];
        Self { pokemons }
    }
}
```

And so yes, it is the moment where we'll (at last) create the `Pokemon` entity in `entities.rs`.

```
pub struct Pokemon {
    pub number: PokemonNumber,
    name: PokemonName,
    types: PokemonTypes,
}

impl Pokemon {
    pub fn new(number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Self {
        Self {
            number,
            name,
            types
        }
    }
}
```

We'll need to make the `entities` module public:

```
// domain/mod.rs
pub mod entities;
```

Now the only compile error we get is the missing `insert` function on `InMemoryRepository`. As we want to be able to insert a Pokemon in any repository implementing the trait `Repository`, we'll add its signature to the trait. For now I'll create the simplest function that makes the code compile:

```
use crate::domain::entities::{Pokemon, PokemonName, PokemonNumber, PokemonTypes};

pub trait Repository {
    fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> PokemonNumber;
}

impl Repository for InMemoryRepository {
    fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> PokemonNumber {
        number
    }
}
```

Let's run the tests:

```
cargo test
running 3 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... FAILED
```

The last test fails. We need to implement (for real this time) the `insert` function. It will be able to return either the number if the insertion succeeds, or a Conflict error if the number already exists. We'll then add an `Insert` struct which will model the return of the `insert` function:

```
pub enum Insert {
    Ok(PokemonNumber),
    Conflict,
}

pub trait Repository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert;
}
```

We can now edit the `insert` function of `InMemoryRepository`:

```
impl Repository for InMemoryRepository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
        if self.pokemons.iter().any(|pokemon| pokemon.number == number) {
            return Insert::Conflict;
        }

        let number_clone = number.clone();
        self.pokemons.push(Pokemon::new(number_clone, name, types));
        Insert::Ok(number)
    }
}
```

For the cloning and the comparison to work, we'll add the attributes on `PokemonNumber` in `domain/entities.rs`:

```
use std::cmp::PartialEq;

#[derive(PartialEq, Clone)]
pub struct PokemonNumber(u16);
```

Now the repository should send back the result of the insert. Let's now use the repository in the use case to make our new test pass ;)

```
fn execute(repo: &mut dyn Repository, req: Request) -> Response {
    match (
        PokemonNumber::try_from(req.number),
        PokemonName::try_from(req.name),
        PokemonTypes::try_from(req.types),
    ) {
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Response::Ok(u16::from(number)),
            Insert::Conflict => Response::Conflict,
        },
        _ => Response::BadRequest,
    }
}
```

Let's run the tests:

```
cargo test
running 3 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
```

Great, our conflict test pass!

## You thought we were done?

Nope. We still have one issue which could happen in the repository. Let say the repository does not work because of an unexpected error. That is a connection error if it is a database, or a network error if it is an API. We should also handle this case.

You know what I'll do now: write a test!

```
#[test]
fn it_should_return_an_error_when_an_unexpected_error_happens() {
    let mut repo = InMemoryRepository::new().with_error();
    let number = 25;
    let req = Request {
        number,
        name: String::from("Pikachu"),
        types: vec![String::from("Electric")],
    };

    let res = execute(&mut repo, req);

    match res {
        Response::Error => {}
        _ => unreachable!(),
    };
}
```

There are two differences with the ok test. First, we add a function to `InMemoryRepository` to make it error out. Second, we check that the `Error` response variant is returned.

Let's first create this new variant:

```
enum Response {
    ...
    Error,
}
```

Now we have to implement the `with_error` function. The idea here is to add a boolean `error` flag to `InMemoryRepository` that we will check at the beginning of each repository function. If the flag is true, we send back an error. Else, we return the usual result. Let's do this in `repositories/pokemon`:

```
pub enum Insert {
    ...
    Error,
}

pub struct InMemoryRepository {
    error: bool,
    pokemons: Vec<Pokemon>,
}

impl InMemoryRepository {
    pub fn new() -> Self {
        let pokemons: Vec<Pokemon> = vec![];
        Self {
            error: false,
            pokemons,
        }
    }

    pub fn with_error(self) -> Self {
        Self {
            error: true,
            ..self
        }
    }
}

impl Repository for InMemoryRepository {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
        if self.error {
            return Insert::Error;
        }

        if self.pokemons.iter().any(|pokemon| pokemon.number == number) {
            return Insert::Conflict;
        }

        let number_clone = number.clone();
        self.pokemons.push(Pokemon::new(number_clone, name, types));
        Insert::Ok(number)
    }
}
```

Now we have just to edit the use case to make it return the corresponding response variant:

```
fn execute(repo: &mut dyn Repository, req: Request) -> Response {
    match (
        PokemonNumber::try_from(req.number),
        PokemonName::try_from(req.name),
        PokemonTypes::try_from(req.types),
    ) {
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Response::Ok(u16::from(number)),
            Insert::Conflict => Response::Conflict,
            Insert::Error => Response::Error,
        },
        _ => Response::BadRequest,
    }
}
```

Let's run the tests:

```
cargo test
running 4 tests
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_an_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

\o/

## Next steps

I feel the length of this article is enough, so let's stop here. Next time, I'll implement the front-end part as an HTTP API. After that, I'll work on the other use cases. I'll also work on having more repository and front-end implementations which will be activated with command line flags. If you want to be updated about the new articles, the blog has an rss feed ;)

Like before, I've created a branch with all the changes on [github](https://github.com/alexislozano/pokedex/tree/article-2).
