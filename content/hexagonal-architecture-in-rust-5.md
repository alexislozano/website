+++
title = "Hexagonal architecture in Rust #5 - Remaining use-cases"
date = 2021-09-12
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1 - Domain](@/hexagonal-architecture-in-rust-1.md)
- [Hexagonal architecture in Rust #2 - In-memory repository](@/hexagonal-architecture-in-rust-2.md)
- [Hexagonal architecture in Rust #3 - HTTP API](@/hexagonal-architecture-in-rust-3.md)
- [Hexagonal architecture in Rust #4 - Refactoring](@/hexagonal-architecture-in-rust-4.md)
- Hexagonal architecture in Rust #5 - Remaining use-cases
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](@/hexagonal-architecture-in-rust-7.md)

Last time, we've done a bit a refactoring. Somehow, our client is angry at us... I mean, he should be happy, the code is now cleaner as ever. 'Kay, he does not know the good things. He's talking about use cases and how we still have work to do.

After having eaten a slice of cake (we deserve it, clearly), let's go back to implementing the remaining use cases. This article will probably be a bit long but, hey, the code is on github if you don't want to read about the process ;) The use cases we'll work on are:

- Fetch all Pokemons
- Fetch a Pokemon
- Delete a Pokemon

## Fetch all Pokemons

As usual, we 'll begin with tests. Let's first create a new use case file: `domain/fetch_all_pokemons.rs`. We'll need some module magic:

```
// domain/mod.rs
pub mod fetch_all_pokemons;
```

Let's see, what are the possibilities for this use case? Pretty simple: everything is successful and we get our Pokemons, or an unknown error occurs in the repository and we cannot get our Pokemons.

Let's begin with the error case with a test:

```
#[cfg(test)]
mod tests {
    use super::*;
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());

        let res = execute(repo);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }
}

```

As you can see, I still did not write one line of code, only the tests. Of course, it won't compile, but we'll see that in time. Two things which are interesting here:

- the test is almost the same as in `domain/create_pokemon.rs`
- we don't need a request

Let's make this test compile :) First we can create our `Error` type:

```
pub enum Error {
    Unknown,
}
```

And then, we can write the simplest function which make the compiler happy:

```
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn execute(repo: Arc<dyn Repository>) -> Result<(), Error> {
    Err(Error::Unknown)
}
```

It compiles and the test pass! Let's now write the next test:

```
#[test]
fn it_should_return_all_the_pokemons_ordered_by_increasing_number_otherwise() {
    let repo = Arc::new(InMemoryRepository::new());
    repo.insert(
        PokemonNumber::pikachu(),
        PokemonName::pikachu(),
        PokemonTypes::pikachu(),
    )
    .ok();
    repo.insert(
        PokemonNumber::charmander(),
        PokemonName::charmander(),
        PokemonTypes::charmander(),
    )
    .ok();

    let res = execute(repo);

    match res {
        Ok(res) => {
            assert_eq!(res[0].number, u16::from(PokemonNumber::charmander()));
            assert_eq!(res[0].name, String::from(PokemonName::charmander()));
            assert_eq!(res[0].types, Vec::<String>::from(PokemonTypes::charmander()));
            assert_eq!(res[1].number, u16::from(PokemonNumber::pikachu()));
            assert_eq!(res[1].name, String::from(PokemonName::pikachu()));
            assert_eq!(res[1].types, Vec::<String>::from(PokemonTypes::pikachu()));
        }
        _ => unreachable!(),
    };
}
```

I've inserted the Pokemons in decreasing number order, so we are sure the repository orders the Pokemons. As you can see, when the repository does not output an error, we get from the use case a vector of responses. Let's first add that to the code:

```
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}

pub fn execute(repo: Arc<dyn Repository>) -> Result<Vec<Response>, Error> {
```

We also have to import our entities in the tests module:

```
use crate::domain::entities::{PokemonName, PokemonNumber, PokemonTypes};
```

Last thing we have to do to make the code compile is to create the missing `charmander` function in the `PokemonNumber` test impl block:

```
pub fn charmander() -> Self {
    Self(4)
}
```

Now the code compiles, but obviously the second test does not pass. Let's implement the `execute` function a bit better:

```
use crate::repositories::pokemon::{FetchAllError, Repository};

pub fn execute(repo: Arc<dyn Repository>) -> Result<Vec<Response>, Error> {
    match repo.fetch_all() {
        Ok(pokemons) => Ok(pokemons
            .into_iter()
            .map(|p| Response {
                number: u16::from(p.number),
                name: String::from(p.name),
                types: Vec::<String>::from(p.types),
            })
            .collect::<Vec<Response>>()),
        Err(FetchAllError::Unknown) => Err(Error::Unknown),
    }
}
```

It won't compile because `fetch_all` and `FetchAllError` do not exist in `Repository`. Let's add them in `repositories/pokemon.rs`.

```
pub enum FetchAllError {
    Unknown,
}

pub trait Repository: Send + Sync {
    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError>;
}

impl Repository for InMemoryRepository {
    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError> {
        if self.error {
            return Err(FetchAllError::Unknown);
        }

        let lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(FetchAllError::Unknown),
        };

        let mut pokemons = lock.to_vec();
        pokemons.sort_by(|a, b| a.number.cmp(&b.number));
        Ok(pokemons)
    }
}
```

Almost done! For `cmp` to work with `PokemonNumber`, we'll have to add some derive attributes:

```
use std::cmp::{..., PartialOrd};

#[derive(..., PartialOrd, Ord, Eq)]
pub struct PokemonNumber(u16);
```

Let's run `cargo test`:

```
cargo test
running 6 tests
...
it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
it_should_return_all_the_pokemons_ordered_by_increasing_number_otherwise ... ok
```

We're done with this use case :) Now let's quickly implement the api. In `api/mod.rs`, add the following:

```
mod fetch_all_pokemons;

// in the router macro
(GET) (/) => {
    fetch_all_pokemons::serve(repo.clone())
}
```

And we can create `api/fetch_all_pokemons.rs` with the following content:

```
use crate::api::Status;
use crate::domain::fetch_all_pokemons;
use crate::repositories::pokemon::Repository;
use rouille;
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(repo: Arc<dyn Repository>) -> rouille::Response {
    match fetch_all_pokemons::execute(repo) {
        Ok(res) => rouille::Response::json(
            &res.into_iter()
                .map(|p| Response {
                    number: p.number,
                    name: p.name,
                    types: p.types,
                })
                .collect::<Vec<Response>>(),
        ),
        Err(fetch_all_pokemons::Error::Unknown) => {
            rouille::Response::from(Status::InternalServerError)
        }
    }
}
```

Now you can run `cargo run`, and open your favorite HTTP client (curl, postman, ...). Create some Pokemons by doing POST requests on `/`. Then you can use your web browser to go to [http://localhost:8000](http://localhost:8000). Here's what I get:

```
[
  {
    "number": 4,
    "name": "Charmander",
    "types": [
      "Fire"
    ]
  },
  {
    "number": 25,
    "name": "Pikachu",
    "types": [
      "Electric"
    ]
  }
]
```

## Fetch a Pokemon

Second use case! Now we want to fetch one Pokemon by giving the system the number of the Pokemon we want. Let's create a new use case file, `domain/fetch_pokemon.rs`, and publish it:

```
// domain/mod.rs
pub mod fetch_pokemon;
```

Let's think about what the use case could return. First, there could be an unkwown error raised by the repository. Second, the request could be wrong. Third, the number we give to the use case could correspond to no Pokemon. Last, we could get the Pokemon with no error.

Let's begin with the unknown error case. We'll open `domain/fetch_pokemon.rs` and add the following to it:

```
use crate::domain::entities::PokemonNumber;
use std::sync::Arc;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());
        let req = Request::new(PokemonNumber::pikachu());
        
        let res = execute(repo, req);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }
}
```

As usual, it does not compile because some types and functions still do not exist. No problem, we'll do it. Let's first begin with the types:

```
pub struct Request {
    pub number: u16,
}

pub enum Error {
    Unknown
}
```

To make the tests clearer, I have added a `new` function to `Request`. Of course, we want to use it in tests only:

```
#[cfg(test)]
mod tests {
    ...

    impl Request {
        fn new(number: PokemonNumber) -> Self {
            Self {
                number: u16::from(number),
            }
        }
    }
}
```

Great, now we only need an execute function. Let's create the simplest function we can write to satisfy this first test:

```
use crate::repositories::pokemon::Repository;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    Err(Error::Unknown)
}
```

Let's run `cargo test fetch_pokemon`:

```
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
```

Nice. We can begin our next test. Let's go with the case where the request is bad formed:

```
#[test]
fn it_should_return_a_bad_request_error_when_request_is_invalid() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request::new(PokemonNumber::bad());

    let res = execute(repo, req);

    match res {
        Err(Error::BadRequest) => {}
        _ => unreachable!(),
    };
}
```

Let's first create the `bad` function in `PokemonNumber`. For that, we can add the following in `domain/entities.rs`:

```
#[cfg(test)]
impl PokemonNumber {
    ...
    pub fn bad() -> Self {
        Self(0)
    }
}
```

Now we can add the new `BadRequest` variant to `Error`:

```
pub enum Error {
    BadRequest,
    ...
}
```

It compiles but the test does not pass. Actually, our `execute` function always returns an unknown error so that's logical :p Let's edit it a bit then:

```
use std::convert::TryFrom;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    match PokemonNumber::try_from(req.number) {
        Ok(number) => Err(Error::Unknown),
        _ => Err(Error::BadRequest),
    }
}
```

Both tests now pass! We can now work on the third test: when the Pokemon is not found in the repository:

```
#[test]
fn it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon() {
    let repo = Arc::new(InMemoryRepository::new());
    let req = Request::new(PokemonNumber::pikachu());

    let res = execute(repo, req);

    match res {
        Err(Error::NotFound) => {}
        _ => unreachable!(),
    };
}
```

Let's add the `NotFound` variant to `Error`:

```
pub enum Error {
    ...
    NotFound,
    ...
}
```

Like before, it compiles but the test does not pass. Let's edit the `Ok` part of the `execute` function:

```
use crate::repositories::pokemon::{FetchOneError, ...};

Ok(number) => match repo.fetch_one(number) {
    Ok(_) => Ok(()),
    Err(FetchOneError::NotFound) => Err(Error::NotFound),
    Err(FetchOneError::Unknown) => Err(Error::Unknown),
}
```

We only need to create `FetchOneError` and `fetch_one` in `repositories/pokemon.rs`:

```
pub enum FetchOneError {
    NotFound,
    Unknown,
}

pub trait Repository: Send + Sync {
    ...
    fn fetch_one(&self, number: PokemonNumber) -> Result<(), FetchOneError>;
}

impl Repository for InMemoryRepository {
    ...
    fn fetch_one(&self, number: PokemonNumber) -> Result<(), FetchOneError> {
        if self.error {
            return Err(FetchOneError::Unknown);
        }

        Err(FetchOneError::NotFound)
    }
}
```

Et voilà, the test pass! That's great and all but we still don't get the Pokemon on success. Let's create another test for this case:

```
#[cfg(test)]
mod tests {
    use crate::domain::entities::{PokemonName, PokemonTypes};

    #[test]
    fn it_should_return_the_pokemon_otherwise() {
        let repo = Arc::new(InMemoryRepository::new());
        repo.insert(
            PokemonNumber::pikachu(),
            PokemonName::pikachu(),
            PokemonTypes::pikachu(),
        )
        .ok();
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Ok(res) => {
                assert_eq!(res.number, u16::from(PokemonNumber::pikachu()));
                assert_eq!(res.name, String::from(PokemonName::pikachu()));
                assert_eq!(res.types, Vec::<String>::from(PokemonTypes::pikachu()));
            }
            _ => unreachable!(),
        };
    }
}
```

Let's first create a `Response` type and edit the `execute` function signature:

```
pub struct Response {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<Response, Error> {
```

Okay, now we can edit the `execute` function `Ok(_)` part:

```
use crate::domain::entities::{Pokemon, ...};

Ok(Pokemon {
    number,
    name,
    types,
}) => Ok(Response {
    number: u16::from(number),
    name: String::from(name),
    types: Vec::<String>::from(types),
})
```

Only remain some edits to the `fetch_one` repo function:

```
pub trait Repository: Send + Sync {
    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError>;
}

impl Repository for InMemoryRepository {
    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
        if self.error {
            return Err(FetchOneError::Unknown);
        }

        let lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(FetchOneError::Unknown),
        };

        match lock.iter().find(|p| p.number == number) {
            Some(pokemon) => Ok(pokemon.clone()),
            None => Err(FetchOneError::NotFound),
        }
    }
}
```

All tests should pass now:

```
test it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon ... ok
test it_should_return_the_pokemon_otherwise ... ok
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
```

Let's now edit the API to make it use this use case. Let's first add the route in `api/mod.rs`:

```
mod fetch_pokemon;

// in the router macro
(GET) (/{number: u16}) => {
    fetch_pokemon::serve(repo.clone(), number)
}
```

We will now create `api/fetch_pokemon.rs` where we will add the following:

```
use crate::api::Status;
use crate::domain::fetch_pokemon;
use crate::repositories::pokemon::Repository;
use rouille;
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(repo: Arc<dyn Repository>, number: u16) -> rouille::Response {
    let req = fetch_pokemon::Request { number };
    match fetch_pokemon::execute(repo, req) {
        Ok(fetch_pokemon::Response {
            number,
            name,
            types,
        }) => rouille::Response::json(&Response {
            number,
            name,
            types,
        }),
        Err(fetch_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(fetch_pokemon::Error::NotFound) => rouille::Response::from(Status::NotFound),
        Err(fetch_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

Now you can `cargo run` the application, open your HTTP client and add some Pokemons to the repository. You should then be able to get them one by one by adding the number of your Pokemon at the end of the url. For example, on my computer, a GET request on [http://localhost:8000/25](http://localhost:8000/25) gives me:

```
{
  "number": 25,
  "name": "Pikachu",
  "types": [
    "Electric"
  ]
}
```

## Delete a Pokemon

I promise, we'll be done soon. Deleting a Pokemon is our last use case. We have four possibilities for what we can get from the use case:

- Success
- Bad request error
- Not found error
- Unknown error

As you've probably noted, they are exactly the same as in the fetch a Pokemon use case. I will be faster in my explanation then. And it begins now. Let's directly write all the tests:

```
// domain/mod.rs
pub mod delete_pokemon;

// domain/delete_pokemon.rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::domain::entities::{PokemonName, PokemonTypes};
    use crate::repositories::pokemon::InMemoryRepository;

    #[test]
    fn it_should_return_an_unknown_error_when_an_unexpected_error_happens() {
        let repo = Arc::new(InMemoryRepository::new().with_error());
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Err(Error::Unknown) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_bad_request_error_when_request_is_invalid() {
        let repo = Arc::new(InMemoryRepository::new());
        let req = Request::new(PokemonNumber::bad());

        let res = execute(repo, req);

        match res {
            Err(Error::BadRequest) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon() {
        let repo = Arc::new(InMemoryRepository::new());
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Err(Error::NotFound) => {}
            _ => unreachable!(),
        };
    }

    #[test]
    fn it_should_return_ok_otherwise() {
        let repo = Arc::new(InMemoryRepository::new());
        repo.insert(
            PokemonNumber::pikachu(),
            PokemonName::pikachu(),
            PokemonTypes::pikachu(),
        )
        .ok();
        let req = Request::new(PokemonNumber::pikachu());

        let res = execute(repo, req);

        match res {
            Ok(()) => {},
            _ => unreachable!(),
        };
    }

    impl Request {
        fn new(number: PokemonNumber) -> Self {
            Self {
                number: u16::from(number),
            }
        }
    }
}
```

Apart from the successful test, it is basically a copy-paste of the tests of `domain/fetch_pokemon.rs`. Next are the types:

```
pub struct Request {
    pub number: u16,
}

pub enum Error {
    BadRequest,
    NotFound,
    Unknown,
}
```

We don't need a `Response` type as we won't be returning anything in the `Ok` case. Let's define the `execute` function:

```
use crate::domain::entities::PokemonNumber;
use crate::repositories::pokemon::{DeleteError, Repository};
use std::convert::TryFrom;
use std::sync::Arc;

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Result<(), Error> {
    match PokemonNumber::try_from(req.number) {
        Ok(number) => match repo.delete(number) {
            Ok(()) => Ok(()),
            Err(DeleteError::NotFound) => Err(Error::NotFound),
            Err(DeleteError::Unknown) => Err(Error::Unknown),
        },
        _ => Err(Error::BadRequest),
    }
}
```

Great! Now we only have to edit the repository. We'll also do it quickly:

```
pub enum DeleteError {
    NotFound,
    Unknown,
}

pub trait Repository: Send + Sync {
    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError>;
}

impl Repository for InMemoryRepository {
    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
        if self.error {
            return Err(DeleteError::Unknown);
        }

        let mut lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Err(DeleteError::Unknown),
        };

        let index = match lock.iter().position(|p| p.number == number) {
            Some(index) => index,
            None => return Err(DeleteError::NotFound),
        };

        lock.remove(index);
        Ok(())
    }
}
```

And now all four tests pass:

```
test it_should_return_a_bad_request_error_when_request_is_invalid ... ok
test it_should_return_a_not_found_error_when_the_repo_does_not_contain_the_pokemon ... ok
test it_should_return_an_unknown_error_when_an_unexpected_error_happens ... ok
test it_should_return_ok_otherwise ... ok
```

We can now add the new route in our API. Let's add the following to `api/mod.rs`:

```
mod delete_pokemon;

// in the router macro
(DELETE) (/{number: u16}) => {
    delete_pokemon::serve(repo.clone(), number)
}

enum Status {
    Ok,
    ...
}

impl From<Status> for rouille::Response {
    fn from(status: Status) -> Self {
        let status_code = match status {
            Status::Ok => 200,
            ...
```

I've added a case for a 200 HTTP response with an empty body. We can now create `api/delete_pokemon.rs`:

```
use crate::api::Status;
use crate::domain::delete_pokemon;
use crate::repositories::pokemon::Repository;
use rouille;
use std::sync::Arc;

pub fn serve(repo: Arc<dyn Repository>, number: u16) -> rouille::Response {
    let req = delete_pokemon::Request { number };
    match delete_pokemon::execute(repo, req) {
        Ok(()) => rouille::Response::from(Status::Ok),
        Err(delete_pokemon::Error::BadRequest) => rouille::Response::from(Status::BadRequest),
        Err(delete_pokemon::Error::NotFound) => rouille::Response::from(Status::NotFound),
        Err(delete_pokemon::Error::Unknown) => rouille::Response::from(Status::InternalServerError),
    }
}
```

By using your HTTP client, you should now be able to delete Pokemons :)

## Conclusion

I hope it wasn't too long... But now our client is happy! Yay! Next time, we'll implement another front-end to our use cases. For now we only have a HTTP API, it would be nice to have a CLI to manage our Pokemons ;) As usual, if you want to be notified when a new article is released, you can subscribe to the rss feed of my blog or to my twitter.

The code is accessible on [github](https://github.com/alexislozano/pokedex/tree/article-5).
