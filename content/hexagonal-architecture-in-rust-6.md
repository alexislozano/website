+++
title = "Hexagonal architecture in Rust #6"
date = 2021-10-09
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1](@/hexagonal-architecture-in-rust-1.md)
- [Hexagonal architecture in Rust #2](@/hexagonal-architecture-in-rust-2.md)
- [Hexagonal architecture in Rust #3](@/hexagonal-architecture-in-rust-3.md)
- [Hexagonal architecture in Rust #4](@/hexagonal-architecture-in-rust-4.md)
- [Hexagonal architecture in Rust #5](@/hexagonal-architecture-in-rust-5.md)
- Hexagonal architecture in Rust #6

Hey, long time no see! Don't be afraid, I haven't forgotten this series. Last time, we implemented the remaining use cases and we have connected them to our API. Now, I want to add another way to use our program. We'll go with a CLI. CLI means Command Line Interface, it is just an acronym to say: "Let's use this program through the terminal".

## Build the scaffolding

Building a CLI means adding new libraries and a new folder in our project. Let's begin with the dependencies. We need a way to:
- choose between running the CLI or the API
- be prompted about what we want to do

To do that, we'll use two crates. To choose between running the CLI or an API, we'll want to add a command line flag. For this we'll use [clap](https://crates.io/crates/clap). To easily create the prompts, we'll also use a crate named [dialoguer](https://crates.io/crates/dialoguer).

Let's add them to the project! Open `Cargo.toml` and add:

```
[dependencies]
...
clap = "2.33.3"
dialoguer = "0.8.0"
```

You can change the versions to the latest ones when you read this article obviously.

Now, we will add the flag. Open `main.rs`. In it we'll first tell our program to use clap:

```
#[macro_use]
extern crate clap;

use clap::{App, Arg};
```

And then we can use it:

```
fn main() {
    let repo = Arc::new(InMemoryRepository::new());

    let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .arg(Arg::with_name("cli").long("cli").help("Runs in CLI mode"))
        .get_matches();

    match matches.occurrences_of("cli") {
        0 => api::serve("localhost:8000", repo),
        _ => unreachable!(),
    }
}
```

So, first we create to repo. Then we create a clap `App` which will handle the CLI. By adding the `--cli` flag, the program will crash for now. Without it, the API runs.

As I've said before, clap let us create a simple CLI. You can try it by running:

```
cargo run -- --help

pokedex 0.1.0
Alexis Lozano <alexis.pascal.lozano@gmail.com>

USAGE:
    pokedex [FLAGS]

FLAGS:
        --cli        Runs in CLI mode
    -h, --help       Prints help information
    -V, --version    Prints version information
```

Pretty nice, uh?

Let's now replace this ugly `unreachable!()` by `cli::run(repo)`. `cli` will be our new module where all the CLI only code will live. We can add the import to `main.rs`. This won't compile for now, but hey, bear with me.

```
mod cli;
```

Now we are ready to create the cli module! Let's create a `cli` folder in `src` and add a `mod.rs` file in it. Add the following code in `cli/mod.rs`:

```
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {}
```

This won't do anything but at least, it should compile and you should be able to run `cargo run -- --cli` without an issue.

Let's now create the prompt loop:

```
use dialoguer::{theme::ColorfulTheme, Select};

pub fn run(repo: Arc<dyn Repository>) {
    loop {
        let choices = [
            "Fetch all Pokemons",
            "Fetch a Pokemon",
            "Create a Pokemon",
            "Delete a Pokemon",
            "Exit",
        ];
        let index = match Select::with_theme(&ColorfulTheme::default())
            .with_prompt("Make your choice")
            .items(&choices)
            .default(0)
            .interact()
        {
            Ok(index) => index,
            _ => continue,
        };

        match index {
            4 => break,
            _ => continue,
        };
    }
}
```

Here we display every command the user can run. If the user chooses the `Exit` command, the program will exit. Otherwise, the program doesn't do anything. Don't worry, we'll now implement each command.

## Create a Pokemon

Let's begin with creating a Pokemon. It'll be easier to test the other commands afterwards if we have a way to fill the repository. This command corresponds to choice number 2, let's add it to `match index`:

```
match index {
    2 => create_pokemon::run(repo.clone()),
    ...
};
```

We now have to create a new module. Let's add it at the top of `cli/mod.rs`:

```
mod create_pokemon;
```

Okay, now we can create `cli/create_pokemon.rs` and add the following:

```
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {}
```

To create a Pokemon, the CLI will ask us the number, the name, and the types of the Pokemon. To easy this process, and as these prompts could be reused in other use cases, we will implement some prompt functions. Let's first use them as if they were existing:

```
use crate::cli::{prompt_name, prompt_number, prompt_types};

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();
    let name = prompt_name();
    let types = prompt_types();
}
```

Now, we'll implement then in `cli/mod.rs`:

```
use dialoguer::{..., Input, MultiSelect};

pub fn prompt_number() -> Result<u16, ()> {
    match Input::new().with_prompt("Pokemon number").interact_text() {
        Ok(number) => Ok(number),
        _ => Err(()),
    }
}

pub fn prompt_name() -> Result<String, ()> {
    match Input::new().with_prompt("Pokemon name").interact_text() {
        Ok(name) => Ok(name),
        _ => Err(()),
    }
}

pub fn prompt_types() -> Result<Vec<String>, ()> {
    let types = ["Electric", "Fire"];
    match MultiSelect::new()
        .with_prompt("Pokemon types")
        .items(&types)
        .interact()
    {
        Ok(indexes) => Ok(indexes
            .into_iter()
            .map(|index| String::from(types[index]))
            .collect::<Vec<String>>()),
        _ => Err(()),
    }
}
```

As you can see, all the prompts can fail. Let's come back to `cli/create_pokemon.rs`. With the values we get from the user, we can create a use case request struct: 

```
use crate::domain::create_pokemon;

pub fn run(repo: Arc<dyn Repository>) {
    ...
    let req = match (number, name, types) {
        (Ok(number), Ok(name), Ok(types)) => create_pokemon::Request {
            number,
            name,
            types,
        },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
```

In case of an input error, we come back to the main menu. Ok, so now we have a request. We should be able to give it to our create pokemon use case:

```
pub fn run(repo: Arc<dyn Repository>) {
    ...
    match create_pokemon::execute(repo, req) {
        Ok(res) => {},
        Err(create_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(create_pokemon::Error::Conflict) => println!("The Pokemon already exists"),
        Err(create_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    };
}
```

We have handled all the error cases with messages the user will be able to read. Now, when the use case succeeds, we want to display a response. Let's do that:

```
#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    ...
    match create_pokemon::execute(repo, req) {
        Ok(res) => println!(
            "{:?}",
            Response {
                number: res.number,
                name: res.name,
                types: res.types,
            }
        ),
        ...
    };
}
```

Okay, let's test that! Go to your terminal, run the program in CLI mode, choose the `Create a Pokemon` command and input the values you want:

```
✔ Make your choice · Create a Pokemon
Pokemon number: 25
Pokemon name: Pikachu
Pokemon types: Electric
Response { number: 25, name: "Pikachu", types: ["Electric"] }
```

Great! First command implemented, let's do the next one!

## Fetch all Pokemons

Now we can create Pokemons, let's fetch them all! For that, we first need to add this case to to `match index` in `cli/mod.rs`:

```
match index {
    0 => fetch_all_pokemons::run(repo.clone()),
    ...
};
```

And we also need to import the soon to exist module:

```
mod fetch_all_pokemons;
```

Now, we can create `cli/fetch_all_pokemons.rs`:

```
use crate::domain::fetch_all_pokemons;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    match fetch_all_pokemons::execute(repo) {
        Ok(res) => res.into_iter().for_each(|p| {
            println!(
                "{:?}",
                Response {
                    number: p.number,
                    name: p.name,
                    types: p.types,
                }
            );
        }),
        Err(fetch_all_pokemons::Error::Unknown) => println!("An unknown error occurred"),
    };
}
```

Yes, everything is here. I explained more precisely the first use case, so I feel I don't need to make baby steps now. So, for this command, we don't need to prompt anything, we just get every Pokemon we can find in the repository. Then we loop in the Response vector and print each one.

## Fetch a Pokemon

Let's now implement the fetch for only on Pokemon. As usual, let's first add the module import and edit `match index` in `cli/mod.rs`:

```
mod fetch_pokemon;

...

match index {
    ...
    1 => fetch_pokemon::run(repo.clone()),
    ...
};
```

And let's create `cli/fetch_pokemon.rs`:

```
use crate::cli::prompt_number;
use crate::domain::fetch_pokemon;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

#[derive(Debug)]
struct Response {
    number: u16,
    name: String,
    types: Vec<String>,
}

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();

    let req = match number {
        Ok(number) => fetch_pokemon::Request { number },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
    match fetch_pokemon::execute(repo, req) {
        Ok(res) => println!(
            "{:?}",
            Response {
                number: res.number,
                name: res.name,
                types: res.types,
            }
        ),
        Err(fetch_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(fetch_pokemon::Error::NotFound) => println!("The Pokemon does not exist"),
        Err(fetch_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    }
}
```

Here, we first ask the number to the user and display an error message if something went wrong. Then we create a Request and send it to the use case. If it succeeds, we print the Response, otherwise we print error messages.

## Delete a Pokemon

Last command! We can add the import and add the command to `match index` in `cli/mod.rs`:

```
mod delete_pokemon;

...

match index {
    ...
    3 => delete_pokemon::run(repo.clone()),
    ...
};
```

Then let's create our new module in `cli/delete_pokemon.rs`:

```
use crate::cli::prompt_number;
use crate::domain::delete_pokemon;
use crate::repositories::pokemon::Repository;
use std::sync::Arc;

pub fn run(repo: Arc<dyn Repository>) {
    let number = prompt_number();

    let req = match number {
        Ok(number) => delete_pokemon::Request { number },
        _ => {
            println!("An error occurred during the prompt");
            return;
        }
    };
    match delete_pokemon::execute(repo, req) {
        Ok(()) => println!("The Pokemon has been deleted"),
        Err(delete_pokemon::Error::BadRequest) => println!("The request is invalid"),
        Err(delete_pokemon::Error::NotFound) => println!("The Pokemon does not exist"),
        Err(delete_pokemon::Error::Unknown) => println!("An unknown error occurred"),
    }
}
```

It is exactly the same as in `cli/fetch_pokemon.rs` minus the Response we don't need.

## Conclusion

Now you can enjoy your Pokedex without sending HTTP requests directly in your comfy terminal :D But you know what we are lacking? A way to really save our Pokemons. For now, they live in the memory so the Repository is empty each time we run the program. It would be cool to create our Pokemons from the CLI and then fetching them from the API :) Next time will be the last article of this series: implementing a long-lived repository. As usual, if you want to be notified when this last article is released, you can subscribe to the rss feed of my blog or to my twitter.

The code is accessible on [github](https://github.com/alexislozano/pokedex/tree/article-6).