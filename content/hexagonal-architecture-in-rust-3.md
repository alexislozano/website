+++
title = "Hexagonal architecture in Rust #3 - HTTP API"
date = 2021-08-26
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1 - Domain](@/hexagonal-architecture-in-rust-1.md)
- [Hexagonal architecture in Rust #2 - In-memory repository](@/hexagonal-architecture-in-rust-2.md)
- Hexagonal architecture in Rust #3 - HTTP API
- [Hexagonal architecture in Rust #4 - Refactoring](@/hexagonal-architecture-in-rust-4.md)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](@/hexagonal-architecture-in-rust-5.md)
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- [Hexagonal architecture in Rust #7 - Long-lived repositories](@/hexagonal-architecture-in-rust-7.md)

Gather, my fellow soldiers. Today we're going to fight! Who, you tell me? The unspoken devil of these lands, the borrow-checker!

Okay, let's stop with this Lord-of-the-Ring-esque impression, work awaits us :) In the previous articles, we defined our domain entities and implemented a use case and a repository.

```
src
├── domain
│   ├── create_pokemon.rs
│   ├── entities.rs
│   └── mod.rs
├── repositories
│   ├── mod.rs
│   └── pokemon.rs
└── main.rs
```

We could have given it to our client, but apart from running the tests, the `main.rs` file still only outputs an hello world. Today, we are going to transform our project into a nice HTTP API returning JSON.

## The HTTP API

If you remember well, I've not used async in the project. This is to focus on the architecture of our application. If you really want to use async, go for it ;) There are not a lot of non-async web framework crates, but some still exists. My choice for this article is [rouille](https://crates.io/crates/rouille), which will handle our use cases nicely.

So first, we open `Cargo.toml` and add it to our dependencies:

```
[dependencies]
rouille = "3.2.1"
```

Let's now create a folder which will contain our API. Inside of it, I'm going to create the usual `mod.rs` file where we'll add the basic routing logic. I'll also add a simple `health.rs` file which will handle our first route:

```
src
└── api
    ├── health.rs
    └── mod.rs
```

The only use of `rouille` will be inside of the `api` folder. If in the future, we want to replace `rouille` with `actix`, we'll just have to change the content of `api` (actually we'll also have to make some functions async but that's orthogonal to the web framework).

Let's now add some code to have a basic working API returning some text when sending a `GET` request on `/health`. For this, let us first import the `rouille` macros in our `main.rs` file and use some function we'll create afterwards:

```
mod api;
mod domain;
mod repositories;

use rouille::router;

fn main() {
    api::serve("localhost:8000");
}
```

Let's now add our `serve` function to the `api/mod.rs`:

```
mod health;

pub fn serve(url: &str) {
    rouille::start_server(url, move |req| {
        router!(req,
            (GET) (/health) => {
                health::serve()
            },
            _ => {
                rouille::Response::from(Status::NotFound)
            }
        )
    });
}
```

I like to create an HTTP status enum to easily return status codes:

```
enum Status {
    BadRequest,
    NotFound,
    Conflict,
    InternalServerError,
}

impl From<Status> for rouille::Response {
    fn from(status: Status) -> Self {
        let status_code = match status {
            Status::BadRequest => 400,
            Status::NotFound => 404,
            Status::Conflict => 409,
            Status::InternalServerError => 500,
        };
        Self {
            status_code,
            headers: vec![],
            data: rouille::ResponseBody::empty(),
            upgrade: None,
        }
    }
}
```

Now, we just have to edit `api/health.rs`:

```
use rouille;

pub fn serve() -> rouille::Response {
    rouille::Response::text("Gotta catch them all!")
}
```

Now you should be able to run the app using `cargo run` and go to [localhost:8000/health](localhost:8000/health) with your browser. There, a beautiful message is waiting for you:

```
Gotta catch them all!
```

Great! But I told before we want a JSON API. Let's convert this endpoint to return JSON then. `serde` will help us. `rouille` already uses some `serde` traits, let's add the same version as `rouille` uses in our `Cargo.toml`. To do that I run `cargo tree | grep serde`:

```
├── serde v1.0.129
├── serde_derive v1.0.129 (proc-macro)
├── serde_json v1.0.66
│   └── serde v1.0.129
```

So let's go with:

```
[dependencies]
rouille = "3.2.1"
serde = { version = "1.0.129", features = ["derive"] }
serde_json = "1.0.66"
```

We are done with the dependencies! Now we can edit `api/health.rs`:

```
use rouille;
use serde::Serialize;

#[derive(Serialize)]
struct Response {
    message: String,
}

pub fn serve() -> rouille::Response {
    rouille::Response::json(&Response {
        message: String::from("Gotta catch them all!"),
    })
}
```

Open your browser and tada :D

```
{
    "message": "Gotta catch them all!"
}
```

## Fetch the request

Our client does not really want a message, even if it's nice. What he wants is being able to create a Pokemon. Let's do it then. First, as our API will be RESTful, here is an example of the HTTP requests we'll work with:

```
- POST http://localhost:8000
- Headers
    Content-Type: application/json
- Body 
    {
        "number": 4,
        "name": "Charmander",
        "types": ["Fire"]
    }
```

Now let's go back to `api/mod.rs` to add a new route:

```
mod create_pokemon;
mod health;

pub fn serve(url: &str) {
    rouille::start_server(url, move |req| {
        router!(req,
            ...
            (POST) (/) => {
                create_pokemon::serve(req)
            },
            ...
        )
    });
}
```

Let's create a new file `api/create_pokemon.rs` and fill it with the following stub:

```
use rouille;
use serde::Serialize;

#[derive(Serialize)]
struct Response {
    message: String,
}

pub fn serve(_req: &rouille::Request) -> rouille::Response {
    rouille::Response::json(&Response {
        message: String::from("Pokemon created!"),
    })
}
```

Now you can use your favorite REST client (postman, curl, ...) to send a POST request on `localhost:8000`, body can be anything for now. You should get back the following:

```
{
    "message": "Pokemon created!"
}
```

That's nice and all, but it would be better to send back a 400 status code when the request body is not what we want. Let's edit `api/create_pokemon.rs` a bit:

```
use crate::api::Status;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Request {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn serve(req: &rouille::Request) -> rouille::Response {
    match rouille::input::json_input::<Request>(req) {
        Ok(_) => {}
        _ => return rouille::Response::from(Status::BadRequest),
    };
    ...
} 
```

Now, if you send a request with no `name` value, or if `number` is negative, you'll get a 400 status code back.

## Plug the repository

Okay, but the Pokemon is actually neither created nor added to a repository. Moreover, the use case is never called! Let's first add an in-memory repository in `main.rs`:

```
use repositories::pokemon::InMemoryRepository;

fn main() {
    let repo = InMemoryRepository::new();
    api::serve("localhost:8000", &repo);
}
```

So we now have to edit accordingly `api/mod.rs`:

```
use crate::repositories::pokemon::Repository;

pub fn serve(url: &str, repo: &mut dyn Repository) {
    rouille::start_server(url, move |req| {
        router!(req,
            ...
            (POST) (/) => {
                create_pokemon::serve(repo, req)
            },
            ...
        )
    });
}
```

Don´t forget to edit `api/create_pokemon.rs`:

```
use crate::repositories::pokemon::Repository;

pub fn serve(_repo: &mut dyn Repository, req: &rouille::Request) -> rouille::Response {
```

You can now run `cargo run` and it should be w...

```
error[E0277]: `dyn Repository` cannot be sent between threads safely
= help: the trait `Send` is not implemented for `dyn Repository`
error[E0277]: `dyn Repository` cannot be shared between threads safely
= help: the trait `Sync` is not implemented for `dyn Repository`
error: aborting due to 2 previous errors
```

I've removed a lot of error log but you have the basics here. Something does not work and it is because of... the borrow-checker. I mean it is because of me, but the borrow-checker got our back :)

## Checkmate for the borrow-checker

As usual, the compiler is quite helpful: it tells us to implement `Send` and `Sync` on `Repository`. Let's do it by editing `repositories/pokemon.rs`:

```
pub trait Repository: Send + Sync {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert;
}
```

Rust is easy, right? Our relief will be very short because as soon as `cargo run` is run:

```
error[E0621]: explicit lifetime required in the type of `repo`
 --> src/api/mod.rs:7:5
  |
6 | pub fn serve(url: &str, repo: &mut dyn Repository) {
  |                               ------------------- help: add explicit lifetime `'static` to the type of `repo`: `&'static mut (dyn Repository + 'static)`
```

Now, the compiler tells us he wants a `'static` lifetime on the repository. Let's think about it for a bit. What is really the issue here? We want a reference to the repository to be sent to the various threads which will be spawned for every request. For now we create a repository with our `InMemory` struct. The thing is, when our app execution will get to the end of the `main` function, `InMemory` will be dropped. But maybe some threads will still have a reference to this struct. Hence the compiler error. What we want instead is some way to tell the app to drop `InMemory` only when the references do not exist anymore. This way is called a reference counter. And we are in luck, Rust provides two types for that, with one especially created to be safe to share between threads. Its name is `Arc`, and that's what we'll be using.

So let's add the `Arc` wrapper to our repository in `main.rs`.

```
use std::sync::Arc;

fn main() {
    let repo = Arc::new(InMemoryRepository::new());
    api::serve("localhost:8000", repo);
}
```

You can see that we removed two things: a `&` and a `mut`. `Arc` is actually a pointer so its size is known at compile time. It points to the repository which is located in the heap. Therefore we don't need a reference to it. Secondly, `Arc` is not mutable so we'll have to use inner mutability. We'll talk about that later.

Let's now edit `api/mod.rs`:

```
use std::sync::Arc;

pub fn serve(url: &str, repo: Arc<dyn Repository>) {
    rouille::start_server(url, move |req| {
        router!(req,
            ...
            (POST) (/) => {
                create_pokemon::serve(repo.clone(), req)
            },
            ...
        )
    });
}
```

And finally, let's edit `api/create_pokemon.rs`:

```
use std::sync::Arc;

pub fn serve(_repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
```

It compiles \o/

## The domain needs some love too

We have designed our app around a domain with use cases getting data and a repository. Like before, we'll have to replace the mutable reference to an `Arc` in the use case too. Good thing I've only implemented one use case for now ;) Let's edit the `execute` function signature in `domain/create_pokemon.rs`:

```
use std::sync::Arc;

fn execute(repo: Arc<dyn Repository>, req: Request) -> Response {
```

Don't forget the tests!

```
let repo = Arc::new(InMemoryRepository::new());
let res = execute(repo, req);
```

On `cargo run`, we stumb upon an issue I've talked about before: an `Arc` is not mutable.

```
25 |         (Ok(number), Ok(name), Ok(types)) => match repo.insert(number, name, types) {
   |                                                    ^^^^ cannot borrow as mutable
```

If we check `Repository` in `repositories/pokemon.rs`, we can actually see that the `insert` function expects the repository to be mutable:

```
pub trait Repository: Send + Sync {
    fn insert(&mut self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert;
}
```

So we'll remove this `mut` in the trait and in our implementation :) Let's run `cargo run`:

```
36 |     fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
   |               ----- help: consider changing this to be a mutable reference: `&mut self`
...
46 |         self.pokemons.push(Pokemon::new(number_clone, name, types));
   |         ^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```

Ouch, this error message is not very helpful. We just removed `mut` and now the compiler wants us to add it back. Actually that's logical, the compiler does not know the repository is inside an `Arc`. 

The interesting thing here is that the issue is not on the trait anymore but only on our `InMemory` implementation. We need to be able to mutate `pokemons` without `self` being mutable. That's interior mutability. And, again, Rust provides some primitives for that! We'll choose the `Mutex` primitive as it is designed for data being shared between threads. So let's wrap `pokemons` into a `Mutex`:

```
use std::sync::Mutex;

pub struct InMemoryRepository {
    error: bool,
    pokemons: Mutex<Vec<Pokemon>>,
}

impl InMemoryRepository {
    pub fn new() -> Self {
        let pokemons: Mutex<Vec<Pokemon>> = Mutex::new(vec![]);
        Self {
            error: false,
            pokemons,
        }
    }
}
```

Now, we'll have to lock the `Mutex` in order to read or write pokemons. Locking a `Mutex` means that the theads wait in turns to read or write the data it holds so only one thread accesses the data at all times.

```
impl Repository for InMemoryRepository {
    fn insert(&self, number: PokemonNumber, name: PokemonName, types: PokemonTypes) -> Insert {
        if self.error {
            return Insert::Error;
        }

        let mut lock = match self.pokemons.lock() {
            Ok(lock) => lock,
            _ => return Insert::Error,
        };

        if lock.iter().any(|pokemon| pokemon.number == number) {
            return Insert::Conflict;
        }

        let number_clone = number.clone();
        lock.push(Pokemon::new(number_clone, name, types));
        Insert::Ok(number)
    }
}
```

Now it compiles and all the tests still pass!

## API + domain = <3

It is the time to connect the API to the domain. Let's edit `api/create_pokemon.rs`:

```
use crate::domain::create_pokemon;

pub fn serve(repo: Arc<dyn Repository>, req: &rouille::Request) -> rouille::Response {
    let req = match rouille::input::json_input::<Request>(req) {
        Ok(req) => create_pokemon::Request {
            number: req.number,
            name: req.name,
            types: req.types,
        },
        _ => return rouille::Response::from(Status::BadRequest),
    };
    match create_pokemon::execute(repo, req) {
        create_pokemon::Response::Ok(number) => rouille::Response::json(&Response { number }),
        create_pokemon::Response::BadRequest => rouille::Response::from(Status::BadRequest),
        create_pokemon::Response::Conflict => rouille::Response::from(Status::Conflict),
        create_pokemon::Response::Error => rouille::Response::from(Status::InternalServerError),
    }
}
```

And publish the needed things from domain:

```
// domain/mod.rs
pub mod create_pokemon;

// domain/create_pokemon.rs
pub struct Request {
    pub number: u16,
    pub name: String,
    pub types: Vec<String>,
}

pub enum Response {
    ...
}

pub fn execute(repo: Arc<dyn Repository>, req: Request) -> Response {
    ...
}
```

You can run `cargo run` and send a valid request to the `create_pokemon` route. You should get a valid response:

```
{
    "number": 30
}
```

\o/

## Next steps

That was longer than expected, sorry for that. I hope it was useful to you :) In the next article, I'll implement the other use cases (the client is tired of waiting for me to explain everything, bad client :p). After that I'll implement other front-end and repositories to get a better understanding of the power of the hexagonal architecture. If you want to be notified of the release of the next article, feel free to subscribe to the rss feed or to my twitter.

As usual, the code is accessible on [github](https://github.com/alexislozano/pokedex/tree/article-3).

## Thanks

- [woshilapin](https://github.com/woshilapin) for fixing some typos
