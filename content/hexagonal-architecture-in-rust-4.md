+++
title = "Hexagonal architecture in Rust #4 - Refactoring"
date = 2021-09-02
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1 - Domain](@/hexagonal-architecture-in-rust-1.md)
- [Hexagonal architecture in Rust #2 - In-memory repository](@/hexagonal-architecture-in-rust-2.md)
- [Hexagonal architecture in Rust #3 - HTTP API](@/hexagonal-architecture-in-rust-3.md)
- Hexagonal architecture in Rust #4 - Refactoring
- [Hexagonal architecture in Rust #5 - Remaining use-cases](@/hexagonal-architecture-in-rust-5.md)
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](@/hexagonal-architecture-in-rust-7.md)

Hi, it is me again! At first I wanted to implement the remaining use cases we still have to work on. But this will be for the next time. Today we will do some refactoring :)

## Before beginning

I've made two changes I did not see in the first articles. In `domain/entities.rs`, I've replaced `u16` with `Self` in:

```
impl From<PokemonNumber> for u16 {
    fn from(n: PokemonNumber) -> Self {
        n.0
    }
}
```

and in `repositories/pokemons.rs`, I've added a test annotation on `with_error`:

```
impl InMemoryRepository {
    #[cfg(test)]
    pub fn with_error(self) -> Self {
        Self {
            error: true,
            ..self
        }
    }
}
```

Let's work on refactoring now ;)

## Use Result instead of custom enums

You read the title, we'll work on the values returned by the functions. We'll do that for the use case and the repository.

### Change the use case return type

First, we'll edit a bit `api/create_pokemon.rs`. We will work with the tests for now, we don't want to have compile errors because the API does not understand we are changing the use case return type.

```
pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    rouille::Response::from(Status::InternalServerError)
    // match create_pokemon::execute(repo, req) {
    //     create_pokemon::Response::Ok(number) => rouille::Response::json(&Response { number }),
    //     create_pokemon::Response::BadRequest => rouille::Response::from(Status::BadRequest),
    //     create_pokemon::Response::Conflict => rouille::Response::from(Status::Conflict),
    //     create_pokemon::Response::Error => rouille::Response::from(Status::InternalServerError),
    // }
}
```

Nice. Let's now go to the `domain/create_pokemon.rs` and change the tested results:

```
    #[test]
    fn it_should_return_a_bad_request_error_when_request_is_invalid() {
        ...
        match res {
            Err(Error::BadRequest) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
        ...
        match res {
            Err(Error::Conflict) => {}
            _ => unreachable!(),
        }
    }

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        ...
        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_the_pokemon_number_otherwise() {
        ...
        match res {
            Ok(res_number) => assert_eq!(res_number, number),
            _ => unreachable!(),
        };
    }
}
```

Great! Now let's create our error type:

```
pub enum Error {
    BadRequest,
    Conflict,
    Unknown,
}
```

We can now edit the use case:

```
pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<u16, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Insert::Ok(number) => Ok(u16::from(number)),
            Insert::Conflict => Err(Error::Conflict),
            Insert::Error => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

and delete the `Response` struct.

Now, let's edit `api/create_pokemons.rs`:

```
pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    match create_pokemon::execute(repo, req) {
        Ok(number) => rouille::Response::json(&Response { number }),
        Err(create_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(create_pokemon::Error::Conflict) => rouille::Response::from(Status::Conflict),
        Err(create_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

The tests should pass now!

```
cargo test
running 4 tests
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

Great! Let's do the same with the repository. 

### Change the repository return types

It does not have tests, so we'll begin by editing the types we get from the repository in our use case:

```
use crate::repositories::pokemon::{InsertError, ...};

...
pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<u16, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Ok(number) => Ok(u16::from(number)),
            Err(InsertError::Conflict) => Err(Error::Conflict),
            Err(InsertError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

In the conflict test, you should add `.ok()` to the repository insert line.

Let's now delete `Insert` and create `InsertError` in `repositories/pokemon.rs`:

```
pub enum InsertError {
    Conflict,
    Unknown,
}
```

We can now change the return types in the functions:

```
pub trait Repository: Send + Sync {
    fn insert(&self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<PokemonNumber, InsertError>;
}

impl Repository for InMemoryRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<PokemonNumber, InsertError> {
        if self.error {
            return Err(InsertError::Unknown);
        }

        let mut lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(InsertError::Unknown),
        };

        if lock.iter().any(|pokemon| pokemon.number == number) {
            return Err(InsertError::Conflict);
        }

        let number_clone = number.clone();
        lock.push(Pokemon::new(number_clone, name, types));
        Ok(number)
    }
}
```

And that's done!

## Return name and types from the create_pokemon use case

In an HTTP API, when creating a new object, I like to get it back. It can also be useful when the created object has fields which were not sent in the request. For example, we could have a `created_at` field which is added by the repository.

First, I let you comment `api/create_pokemon.rs` as we did at the beginning. We'll be able to concentrate on the tests. Talking about tests, let's edit the sucessful test in `domain/create_pokemon.rs`:

```
#[test]
fn it_should_return_the_pokemon_number_otherwise() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request {
        number: 25,
        name: String::from("Pikachu"),
        types: vec![String::from("Electric")],
    };

    let res = execute(repo, req);

    match res {
        Ok(Response {
            number,
            name,
            types,
        }) => {
            assert_eq!(number, 25);
            assert_eq!(name, String::from("Pikachu"));
            assert_eq!(types, vec![String::from("Electric")]);
        }
        _ => unreachable!(),
    };
}
```

Of course, it won't compile because `Response` does not exist yet. Let's create it then:

```
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}
```

Next, we'll edit the `execute` function to make it return a `Response` when successful:

```
use crate::domain::entities::{Pokemon, ...};

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<Response, Error> {
    ...
        (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
            Ok(Pokemon {
                number,
                name,
                types,
            }) => Ok(Response {
                number: u16::from(number),
                name: String::from(name),
                types: Vec::<String>::from(types),
            }),
            Err(InsertError::Conflict) => Err(Error::Conflict),
            Err(InsertError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

As you may have seen, the repository now returns the whole Pokemon on success! Let's edit `repositories/pokemon.rs` then:

```
pub trait Repository: Send + Sync {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError>;
}

impl Repository for InMemoryRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError> {
        ...
        let pokemon = Pokemon::new(number, name, types);
        lock.push(pokemon.clone());
        Ok(pokemon)
    }
}
```

We need to be able to clone a Pokemon. Let's add some `derive` magic in `domain/entities.rs`:

```
#[derive(Clone)]
pub struct PokemonName(String);

#[derive(Clone)]
pub struct PokemonTypes(Vec<PokemonType>);

#[derive(Clone)]
enum PokemonType {

#[derive(Clone)]
pub struct Pokemon {
```

The repository is now working but we still have compilation errors. First, we try to get `name` and `types` from `Pokemon` but these fields are not public. Let's change that:

```
pub struct Pokemon {
    pub number: PokemonNumber,
    pub name: PokemonName,
    pub types: PokemonTypes,
}
```

Second, we try to convert from `PokemonNumber` to `u16`, from `PokemonName` to `String` and from `PokemonTypes` to `Vec<String>`. The first one was already done but the second and third ones are not. Let's do it!

### Pokemon name

```
impl From<PokemonName> for String {
    fn from(n: PokemonName) -> Self {
        n.0
    }
}
```

### Pokemon types

```
impl From<PokemonTypes> for Vec<String> {
    fn from(pts: PokemonTypes) -> Self {
        let mut ts = vec![];
        for pt in pts.0.into_iter() {
            ts.push(String::from(pt));
        }
        ts
    }
}

impl From<PokemonType> for String {
    fn from(t: PokemonType) -> Self {
        String::from(match t {
            PokemonType::Electric => "Electric",
            PokemonType::Fire => "Fire",
        })
    }
}
```

Done! The tests should pass now:

```
cargo test
running 4 tests
test it_should_return_a_conflict_error_when_pokemon_number_already_exists ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_the_pokemon_number_otherwise ... ok
```

We still have to update `api/create_pokemon.rs`. First, we can edit the `Response`:

```
#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}
```

And now we can change the `serve` function:

```
pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = ...
    match create_pokemon::execute(repo, req) {
        Ok(create_pokemon::Response {
            number,
            name,
            types,
        }) => rouille::Response::json(&Response {
            number,
            name,
            types,
        }),
        Err(create_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(create_pokemon::Error::Conflict) => rouille::Response::from(Status::Conflict),
        Err(create_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

When running the server and sending a POST request, we get back the whole Pokemon:

```
{
    "number": 17,
    "name": "Charmander",
    "types": [
        "Fire"
    ]
}
```

## Create tests values

Do you like having things like `PokemonName::try_from(String::from("Pikachu")).unwrap()` in the middle of your tests? Me neither. Let's create some functions in `domain/entities.rs` to help us.

### Pokemon number

```
#[cfg(test)]
impl PokemonNumber {
    pub fn pikachu() -> Self {
        Self(25)
    }
}
```

### Pokemon name

```
#[cfg(test)]
impl PokemonName {
    pub fn pikachu() -> Self {
        Self(String::from("Pikachu"))
    }

    pub fn charmander() -> Self {
        Self(String::from("Charmander"))
    }

    pub fn bad() -> Self {
        Self(String::from(""))
    }
}
```

### Pokemon types

```
#[cfg(test)]
impl PokemonTypes {
    pub fn pikachu() -> Self {
        Self(vec![PokemonType::Electric])
    }

    pub fn charmander() -> Self {
        Self(vec![PokemonType::Fire])
    }
}
```

Let's use all this in the use case tests. Let's first add a function to easily create a `Request` at the end of the tests:

```
#[cfg(test)]
mod tests {
    ...

    impl Request {
        fn new(number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Self {
            Self {
                number: u16::from(number),
                name: String::from(name),
                types: Vec::<String>::from(types),
            }
        }
    }
}
```

And in the test functions:

```
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::bad(),
        PokemonTypes::pikachu(),
    );
    ...
}

#[test]
fn it_should_return_a_conflict_error_when_pokemon_number_already_exists() {
    ...
    repo.insert(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    )
    .ok();
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::charmander(),
        PokemonTypes::charmander(),
    );
    ...
}

#[test]
fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    );
    ...
}

#[test]
fn it_should_return_the_pokemon_otherwise() {
    ...
    let req = Request::new(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    );
    ...    
    match res {
        Ok(res) => {
            assert_eq!(res.number, u16::from(PokemonNumber::pikachu()));
            assert_eq!(res.name, String::from(PokemonName::pikachu()));
            assert_eq!(res.types, Vec::<String>::from(PokemonTypes::pikachu()));
        }
        _ => unreachable!(),
    };
}
```

## Conclusion

We are done with this long refactoring. Hope everything went well ;) I promise, next time we'll implement the other use cases! If you want to be notified of the release of the next article, feel free to subscribe to the rss feed or to my twitter.

As usual, the code is available on [github](https://github.com/alexislozano/pokedex/tree/article-4).

## Thanks

- [woshilapin](https://github.com/woshilapin) for fixing some typos
