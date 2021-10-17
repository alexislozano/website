+++
title = "Hexagonal architecture in Rust #7 - Long-lived repositories"
date = 2021-10-17
+++

This article is part of the following series:

- [Hexagonal architecture in Rust #1 - Domain](@/hexagonal-architecture-in-rust-1.md)
- [Hexagonal architecture in Rust #2 - In-memory repository](@/hexagonal-architecture-in-rust-2.md)
- [Hexagonal architecture in Rust #3 - HTTP API](@/hexagonal-architecture-in-rust-3.md)
- [Hexagonal architecture in Rust #4 - Refactoring](@/hexagonal-architecture-in-rust-4.md)
- [Hexagonal architecture in Rust #5 - Remaining use-cases](@/hexagonal-architecture-in-rust-5.md)
- [Hexagonal architecture in Rust #6 - CLI](@/hexagonal-architecture-in-rust-6.md)
- Hexagonal architecture in Rust #7 - Long-lived repositories

It is now the time for the last article of the series. Thanks a lot to you, anonymous reader who is still here after seven posts. We already have covered a lot of things: we have a domain, we store data, and we can operate our program with both a CLI and an HTTP server. So what do we still have to work on? Oh, I see the client, let's ask him.

- Hello client, how are you?
- I'm okay, but your software is not.
- Oh... What is not working?
- When we reboot the software, we lose all our data.
- Ah, yes that's normal. We are still working directly with volatile memory.
- Do you think you could make the data long-lived?
- The data? Yes. Do you have somewhere where we could store it?
- Hmm, I don't really know. I'll ask someone. Could you make the software use the hard-drive of the machine it is installed on for now?
- I'm on it!

Okay, so we want to store our Pokemons in a place where they won't be deleted after a reboot of the software. You know what? We'll use files :) And because we still want to operate the data in a reliable way, we'll use SQL. Files and SQL... something screams SQLite at the back of my head.

The nice thing here is that we already have a repository system, so we'll just have to add a new repository and some flags in the `main` function to choose which datastore to use. Yup, no change in the domain, that's the beauty of hexagonal architecture :D

## Local datastore with SQLite

### Create the database

First thing we need to do is preparing the database. Let's first create the file using `sqlite3`:

```
sqlite3 path/to/the/database.sqlite
```

You should get back an SQLite prompt. Now we'll want to create two tables. Why two? Because a Pokemon can have an arbitrary number of types. So we cannot store each type in a column. Actually we are in the case of a many-to-many relation:

- One Pokemon can have multiple types
- One type can be set to multiple Pokemon

So we should have the following tables:

```
pokemons: | number | name |

types: | id | name |

pokemons_to_types : | pokemons.number | types.id |
```

But hey, in this case we don't really need type ids, let's use a two tables relational schema:

```
pokemons: | number | name |

types: | pokemons.number | name |
```

That'll do. Let's go back to our SQLite prompt. First we'll activate foreign keys for this connection. Careful: this will only work for one connection. So in our software, we'll have to also run this command:

```
pragma foreign_keys = 1;
```

Done! Now we can create our tables:

```
create table pokemons (
    number integer primary key,
    name text
);

create table types (
    pokemon_number integer,
    name text,
    foreign key (pokemon_number) references pokemons (number) on delete cascade,
    primary key (pokemon_number, name)
);
```

The `on delete cascade` means that on a Pokemon deletion, all types with this pokemon number will be deleted too. Something to not care about in our software, yay ;) You can now exit the SQLite prompt by entering `Ctrl-D`.

### Add flags to choose the datastore

Now we have a datastore, we want to use it. Right now, if you run our program with the `--cli` flag or not, you'll use the in-memory repository. So we'll have to add another flag to tell the program we want to use SQLite. It will be used like so: `pokedex --sqlite path/to/the/database.sqlite`. When using `cargo run` to run the program, the command will be `cargo run -- --sqlite path/to/the/database.sqlite`.

Let's add the flag in `main.rs`:

```
fn main() {
    let repo = Arc::new(InMemoryRepository::new());

    let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .arg(Arg::with_name("cli").long("cli").help("Runs in CLI mode"))
        .arg(Arg::with_name("sqlite").long("sqlite").value_name("PATH"))
        .get_matches();

    match matches.occurrences_of("cli") {
        0 => api::serve("localhost:8000", repo),
        _ => cli::run(repo),
    }
}
```

So now the flag exists but it is not used. You can still see it in the `--help` command:

```
OPTIONS:
        --sqlite <PATH>
```

Nice. Let's now use this flag to initialize a now non-existing `SqliteRepository`:

```
use repositories::pokemon::{..., SqliteRepository};

fn main() {
    let matches = ...

    let repo = build_repo(matches.value_of("sqlite"));

    match matches.occurrences_of("cli") {
        0 => api::serve("localhost:8000", repo),
        _ => cli::run(repo),
    }
}

fn build_repo(sqlite_value: Option<&str>) -> Arc<dyn Repository> {
    if let Some(path) = sqlite_value {
        match SqliteRepository::try_new(path) {
            Ok(repo) => return Arc::new(repo),
            _ => panic!("Error while creating sqlite repo"),
        }
    }

    Arc::new(InMemoryRepository::new())
}
```

As you can see, if the `--sqlite` flag is toggled, the program tries to initialize the SQLite repository. If it cannot, it crashes with an error message. If the flag is not set, it defaults to the in-memory repository. 

### Implementing the repository

I'm talking about SQLite since the beginning of the article. That's great and all, but we need a way to call our database from Rust. That's where [rusqlite](https://crates.io/crates/rusqlite/) enters the game. Let's import it in `Cargo.toml`:

```
[dependencies]
rusqlite = "0.26.0"
```

We now have to implement this repository. Brace yourselves :) First let's go to `repositories/pokemons.rs`. I'll create `SqliteRepository` in the same file as `InMemoryRepository`. That's not mandatory, you can create another file if you prefer. We can first define the struct we'll be using as the repository:

```
use rusqlite::Connection;

pub struct SqliteRepository {
    connection: Mutex<Connection>,
}
```

The `use` statement in `main.rs` should work now. We still have to implement `try_new`:

```
use rusqlite::{..., OpenFlags};

impl SqliteRepository {
    pub fn try_new(path: &str) -> Result<Self, ()> {
        let connection = match Connection::open_with_flags(path, OpenFlags::SQLITE_OPEN_READ_WRITE)
        {
            Ok(connection) => connection,
            _ => return Err(()),
        };

        match connection.execute("pragma foreign_keys = 1", []) {
            Ok(_) => Ok(Self {
                connection: Mutex::new(connection),
            }),
            _ => Err(()),
        }
    }
}
```

So what's this code about? First we ask `rusqlite` to open a connection for us to the file. We've explicitely forbidden `rusqlite` to create the database file using the `OpenFlags`. If the connection is ok, we execute the line I've talked about earlier to make sure the foreign keys work. Then we return a nice `SqliteRepository`.

If you try to run the program now, it won't compile. Why? Because we told in `main.rs` that `SqliteRepository` implements `Repository`. Let's first make the program compile by implementing the most basic working code:

```
impl Repository for SqliteRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError> {
        Err(InsertError::Unknown)
    }

    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError>{
        Err(FetchAllError::Unknown)
    }

    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
        Err(FetchOneError::Unknown)
    }

    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
        Err(DeleteError::Unknown)
    }
}
```

It compiles, but our repository is not really useful. We'll work on each function :)

#### Helper functions

Before implementing each function `Repository` needs, let's create two helper functions. We want to fetch Pokemons is both `fetch_one` and `fetch_all` functions, that's a select in SQL. We are going to create helpers to have the fetching logic in one place.

First, let's work on a function to get back the number and the name of Pokemons. This function will take an already locked connection and maybe a number. If the function takes a number, it will add a `where` clause to the SQL query. We'll add this function in `impl SqliteRepository` because it is not part of the `Repository` trait. Plus, as we'll work with bare values (u16 and String), this function won't be published:

```
use std::sync::{..., MutexGuard};

fn fetch_pokemon_rows(
    lock: &MutexGuard<'_, Connection>,
    number: Option<u16>,
) -> Result<Vec<(u16, String)>, ()> {
    // code will go here
}
```

Okay, let's begin. First we want to define the query and the query parameters depending on the presence of a number:

```
let (query, params) = match number {
    Some(number) => (
        "select number, name from pokemons where number = ?",
        vec![number],
    ),
    _ => ("select number, name from pokemons", vec![]),
};
```

Fairly simple right? Now we'll have to prepare a statement and give it our parameters:

```
use rusqlite::{..., params_from_iter};

...
let mut stmt = match lock.prepare(query) {
    Ok(stmt) => stmt,
    _ => return Err(()),
};

let mut rows = match stmt.query(params_from_iter(params)) {
    Ok(rows) => rows,
    _ => return Err(()),
};
```

We get back rows. We need to convert them to tuples of u16 and String, and put these tuples into a vector we'll return:

```
...
let mut pokemon_rows = vec![];

while let Ok(Some(row)) = rows.next() {
    match (row.get::<usize, u16>(0), row.get::<usize, String>(1)) {
        (Ok(number), Ok(name)) => pokemon_rows.push((number, name)),
        _ => return Err(()),
    };
}

Ok(pokemon_rows)
```

Great! We can now do the same thing with types. We want a function which will take an already locked connection and a number. It will return a vector of strings or an error. These strings will be the types this particular Pokemon has. Same as before, the function will be unpublished and will be part of `impl SqliteRepository`:

```
fn fetch_type_rows(lock: &MutexGuard<'_, Connection>, number: u16) -> Result<Vec<String>, ()> {
    // code will go here
}   
```

Let's prepare a statement and query it:

```
let mut stmt = match lock.prepare("select name from types where pokemon_number = ?") {
    Ok(stmt) => stmt,
    _ => return Err(()),
};

let mut rows = match stmt.query([number]) {
    Ok(rows) => rows,
    _ => return Err(()),
};
```

As in the other function, we get rows. We are going to extract the type names from the rows:

```
...
let mut type_rows = vec![];

while let Ok(Some(row)) = rows.next() {
    match row.get::<usize, String>(0) {
        Ok(name) => type_rows.push(name),
        _ => return Err(()),
    };
}

Ok(type_rows)
```

Aaaand, boom done! Both functions are now implemented. It'll be easier to implement `fetch_one` and `fetch_all` with these :)

#### Fetch a Pokemon

We'll go step by step. We will be working in the following function:

```
fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
    // code will go here
}
```

First, we are going to acquire a lock and fetch the pokemon rows using the function we created before:

```
let lock = match self.connection.lock() {
    Ok(lock) => lock,
    _ => return Err(FetchOneError::Unknown),
};

let mut pokemon_rows = match Self::fetch_pokemon_rows(&lock, Some(u16::from(number.clone()))) {
    Ok(pokemon_rows) => pokemon_rows,
    _ => return Err(FetchOneError::Unknown),
};
```

When the rows are empty, we want to send back a `NotFound` error. Otherwise, we take the first row:

```
...
if pokemon_rows.is_empty() {
    return Err(FetchOneError::NotFound);
}

let pokemon_row = pokemon_rows.remove(0);
```

Nice. We can now get the type rows:

```
...
let type_rows = match Self::fetch_type_rows(&lock, pokemon_row.0) {
    Ok(type_rows) => type_rows,
    _ => return Err(FetchOneError::Unknown),
};
```

We have the number, the name and the types of the Pokemon. Time to wrap up:

```
...
match (
    PokemonNumber::try_from(pokemon_row.0),
    PokemonName::try_from(pokemon_row.1),
    PokemonTypes::try_from(type_rows),
) {
    (Ok(number), Ok(name), Ok(types)) => Ok(Pokemon::new(number, name, types)),
    _ => Err(FetchOneError::Unknown),
}
```

#### Fetch all Pokemons

We'll be working in the following function:

```
fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError> {
    // code will go here
}
```

Let's first acquire the lock and the pokemon rows:

```
let lock = match self.connection.lock() {
    Ok(lock) => lock,
    _ => return Err(FetchAllError::Unknown),
};

let pokemon_rows = match Self::fetch_pokemon_rows(&lock, None) {
    Ok(pokemon_rows) => pokemon_rows,
    _ => return Err(FetchAllError::Unknown),
};
```

You can notice that we now use `fetch_pokemon_rows` with a number set to None as we want all Pokemons. For each pokemon, we are going to fetch the matching type rows. We will then create Pokemons from the number, the name and the types we have. And then we return the list of Pokemons:

```
...
let mut pokemons = vec![];

for pokemon_row in pokemon_rows {
    let type_rows = match Self::fetch_type_rows(&lock, pokemon_row.0) {
        Ok(type_rows) => type_rows,
        _ => return Err(FetchAllError::Unknown),
    };

    let pokemon = match (
        PokemonNumber::try_from(pokemon_row.0),
        PokemonName::try_from(pokemon_row.1),
        PokemonTypes::try_from(type_rows),
    ) {
        (Ok(number), Ok(name), Ok(types)) => Pokemon::new(number, name, types),
        _ => return Err(FetchAllError::Unknown),
    };

    pokemons.push(pokemon);
}

Ok(pokemons)
```

#### Insert a Pokemon

We will be working in the following function:

```
fn insert(
    &self,
    number: PokemonNumber,
    name: PokemonName,
    types: PokemonTypes,
) -> Result<Pokemon, InsertError> {
    // code will go here
}
```

First, we are going to take a lock from the connection. So we'll have the inner `Connection`:

```
let mut lock = match self.connection.lock() {
    Ok(lock) => lock,
    _ => return Err(InsertError::Unknown),
};
```

Now we are going to create a transaction. Why can't we directly execute a command? Because we are going to insert a Pokemon in the `pokemons` table and some types in the `types` database. We want to be sure to revert the changes if one of the insert errors out. With a transaction, you first create your SQL query, and then you commit it. `rusqlite` will automatically revert it if something goes wrong.

```
...
let transaction = match lock.transaction() {
    Ok(transaction) => transaction,
    _ => return Err(InsertError::Unknown),
};
```

Now we're going to add our first command to the transaction. We want to insert a Pokemon. If the Pokemon already exists, we want to fail with the corresponding error:

```
use rusqlite::{..., Error::SqliteFailure, params};

...
match transaction.execute(
    "insert into pokemons (number, name) values (?, ?)",
    params![u16::from(number.clone()), String::from(name.clone())],
) {
    Ok(_) => {}
    Err(SqliteFailure(_, Some(message))) => {
        if message == "UNIQUE constraint failed: pokemons.number" {
            return Err(InsertError::Conflict);
        } else {
            return Err(InsertError::Unknown);
        }
    }
    _ => return Err(InsertError::Unknown),
};
```

Here, we're using the error message sent by `rusqlite` to check if the error is due to a conflicting row. Now the Pokemon is inserted, we can insert the types. We'll execute an insert query for each type in the `PokemonTypes` argument the function got:

```
...
for _type in Vec::<String>::from(types.clone()) {
    if let Err(_) = transaction.execute(
        "insert into types (pokemon_number, name) values (?, ?)",
        params![u16::from(number.clone()), _type],
    ) {
        return Err(InsertError::Unknown);
    }
}
```

And now, we can commit the transaction and return the Pokemon if everything went well.

```
...
match transaction.commit() {
    Ok(_) => Ok(Pokemon::new(number, name, types)),
    _ => Err(InsertError::Unknown),
}
```

#### Delete a Pokemon

Now we know how to add a Pokemon, let's implement the deletion. We're going to work on the following function:

```
fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
    // code will go here
}
```

So first we need to acquire the lock:

```
let lock = match self.connection.lock() {
    Ok(lock) => lock,
    _ => return Err(DeleteError::Unknown),
};
```

When it's done, we just have to execute the SQL request to delete the Pokemon with the number we want.

```
...
match lock.execute(
    "delete from pokemons where number = ?",
    params![u16::from(number)],
) {
    Ok(0) => Err(DeleteError::NotFound),
    Ok(_) => Ok(()),
    _ => Err(DeleteError::Unknown),
}
```

Two things to note here. First we don't have to bother about deleting types as we told SQLite to do it automatically with `on delete cascade` earlier. Second, we are using the number of deleted lines sent back in `Ok` to return a `NotFound` error or not. If this number is 0, it means no lines were deleted so the Pokemon was not in the database.

You should now be able to use your SQLite database as your datastore. And this in both the CLI and the HTTP API. What I like to try to test is to create Pokemons from the CLI and fetch them from the API. Pretty nice to see it in action :)

## Interlude

Oh, I see the client, he's coming towards me.

- Hi Alexis!
- Hi client! I've implemented a long lived repository which store the data on the hard drive of the computer.
- Oh, that's great! Though I've got news about the datastore we will use.
- Nice, that was fast. Is it a PostgreSQL or MySQL server?
- No, my company is webscale.
- Hmm, that reminds me of an [horror story](https://www.youtube.com/watch?v=b2F-DItXtZs). But okay, what technology does your company want to use?
- They want to go with Airtable, it is like Excel but online.
- Oh... I'll see what I can do.

## Webscale datastore with Airtable

### Create the database

Okay so now we have to create another repository. That's life. So we can go to [Airtable website](https://airtable.com/) to see how it works. You'll have to create an account and a new workspace. In this workspace you should have a default table defined. Rename it to `pokemons`. You can change the columns of the table with the following:

- number:
    - select the `Number` type
    - select the `Integer` subtype
    - turn off `Allow negative numbers`
- name:
    - select the `Single line text` type
- types:
    - select the `Multiple select` type
    - add `Electric` and `Fire` in the options

Good news, we can use only one table to modelize our data :)

Now, we want to know how we could work with Airtable in our program. Let's go to the [API docs](https://airtable.com/api). Here you can choose the workspace you want to work with. There, you can see the existing clients. At the time of writing this article, there is none for Rust. So we'll use the other possibility: their HTTP API. We will need an HTTP client to use in our program. We'll use `ureq`, let's add it to `Cargo.toml`:

```
[dependencies]
ureq = { version = "2.2.0", features = ["json"] }
```

The `json` feature will be useful to automatically convert HTTP responses to structs thanks to serde. But before implementing the repository, we still have to find a way to be able to choose Airtable when running our program.

### Add the flag to use the datastore

First let's see what we need to use the API. Here's a request example they give in the docs using curl:

```
$ curl https://api.airtable.com/v0/<WORKSPACE_ID>/pokemons \
-H "Authorization: Bearer <API_KEY>"
```

Ok, so we need to give two values to our program: an `api_key` and a `workspace_id`. Let's add the corresponding flag to our `main.rs`:

```
fn main() {
    let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .arg(Arg::with_name("cli").long("cli").help("Runs in CLI mode"))
        .arg(Arg::with_name("sqlite").long("sqlite").value_name("PATH"))
        .arg(
            Arg::with_name("airtable")
                .long("airtable")
                .value_names(&["API_KEY", "WORKSPACE_ID"]),
        )
        .get_matches();
}
```

We will be able to use the Airtable repository by doing `pokedex --airtable <API_KEY> <WORKSPACE_ID>`. When running the pokedex with cargo run, that'll be `cargo run -- --airtable <API_KEY> <WORKSPACE_ID>`. We can check the flag is taken into account by running the program with the `--help` flag:

```
OPTIONS:
        --airtable <API_KEY> <WORKSPACE_ID>
```

Let's use the values we get to create a non-existing `AirtableRepository`:

```
use clap::{..., Values};
use repositories::pokemon::{AirtableRepository, ...};

fn main() {
    ...
    let repo = build_repo(matches.value_of("sqlite"), matches.values_of("airtable"));
    ...
}

fn build_repo(sqlite_value: Option<&str>, airtable_values: Option<Values>) -> Arc<dyn Repository> {
    if let Some(values) = airtable_values {
        if let [api_key, workspace_id] = values.collect::<Vec<&str>>()[..] {
            match AirtableRepository::try_new(api_key, workspace_id) {
                Ok(repo) => return Arc::new(repo),
                _ => panic!("Error while creating airtable repo"),
            }
        }
    }
    ...
}
```

So now, if the `--airtable` flag is set we use Airtable. Otherwise if the `--sqlite` flag is set we use SQLite. Otherwise we use the in-memory repository. Let's now implement the repository ;)

### Implement the repository

I'll add `AirtableRepository` in `repositories/pokemon.rs`. Like before, you can put it in its own file. Let's first create the struct:

```
pub struct AirtableRepository {
    url: String,
    auth_header: String,
}
```

We still have to implement the `try_new` function. We are going to create the `url` using the `workspace_id` and the `auth_header` using the `api_key`. We'll also make a request to be sure we can connect to our datastore:

```
impl AirtableRepository {
    pub fn try_new(api_key: &str, workspace_id: &str) -> Result<Self, ()> {
        let url = format!("https://api.airtable.com/v0/{}/pokemons", workspace_id);
        let auth_header = format!("Bearer {}", api_key);

        if let Err(_) = ureq::get(&url).set("Authorization", &auth_header).call() {
            return Err(());
        }

        Ok(Self { url, auth_header })
    }
}
```

If you try to compile the source code now, it won't. Why? Because we still need to implement `Repository` for `AirtableRepository`. Let's do that we the simplest code possible:

```
impl Repository for AirtableRepository {
    fn insert(
        &self,
        number: PokemonNumber,
        name: PokemonName,
        types: PokemonTypes,
    ) -> Result<Pokemon, InsertError> {
        Err(InsertError::Unknown)
    }

    fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError> {
        Err(FetchAllError::Unknown)
    }

    fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
        Err(FetchOneError::Unknown)
    }

    fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
        Err(DeleteError::Unknown)
    }
}
```

#### Helper function

You should be used to it by now, we'll create one helper function this time. This function will let us get one or several rows from Airtable. It'll take as arguments `&self` and an optional number. It'll return an error if something went wrong or a json record if it went well. It won't be published and will stay in `impl AirtableRepository`. Thanks to serde we'll be able to automatically convert the json into a struct. Let's first defines the structs:

```
#[derive(Deserialize)]
struct AirtableJson {
    records: Vec<AirtableRecord>,
}

#[derive(Deserialize)]
struct AirtableRecord {
    id: String,
    fields: AirtableFields,
}

#[derive(Deserialize)]
struct AirtableFields {
    number: u16,
    name: String,
    types: Vec<String>,
}
```

And the function:

```
fn fetch_pokemon_rows(&self, number: Option<u16>) -> Result<AirtableJson, ()> {
    // code will go here
}
```

First thing we have to do is to create the url depending on the presence of number:

```
let url = match number {
    Some(number) => format!("{}?filterByFormula=number%3D{}", self.url, number),
    None => format!("{}?sort%5B0%5D%5Bfield%5D=number", self.url),
};
```

The query parameters are a bit weird. First one is a filter on number. Second one is a sort on number. Now we can issue the request using ureq:

```
...
let res = match ureq::get(&url)
    .set("Authorization", &self.auth_header)
    .call()
{
    Ok(res) => res,
    _ => return Err(()),
};
```

We have the HTTP reponse in `res`, we are going to convert it to an `AirtableJson` thanks to serde:

```
...
match res.into_json::<AirtableJson>() {
    Ok(json) => Ok(json),
    _ => Err(()),
}
```

And, that's done. Pretty fast but really nice :)

#### Fetch one Pokemon

Okay, let's implement the following function:

```
fn fetch_one(&self, number: PokemonNumber) -> Result<Pokemon, FetchOneError> {
    // code will go here
}
```

First, we'll get the json using our nice function we implemented before:

```
let mut json = match self.fetch_pokemon_rows(Some(u16::from(number.clone()))) {
    Ok(json) => json,
    _ => return Err(FetchOneError::Unknown),
};
```

Now, if the json does not contains any records, it means the Pokemon we wanted is not present in Airtable. So we'll return an error. Otherwise, we take the first record:

```
...
if json.records.is_empty() {
    return Err(FetchOneError::NotFound);
}

let record = json.records.remove(0);
```

Last thing we need to do is converting this record to a Pokemon:

```
...
match (
    PokemonNumber::try_from(record.fields.number),
    PokemonName::try_from(record.fields.name),
    PokemonTypes::try_from(record.fields.types),
) {
    (Ok(number), Ok(name), Ok(types)) => Ok(Pokemon::new(number, name, types)),
    _ => Err(FetchOneError::Unknown),
}
```

We're good with `fetch_one`!

#### Fetch all Pokemons

We're going to work on this function:

```
fn fetch_all(&self) -> Result<Vec<Pokemon>, FetchAllError> {
    // code will go here
}
```

First, let's get the json using the helper function:

```
let json = match self.fetch_pokemon_rows(None) {
    Ok(json) => json,
    _ => return Err(FetchAllError::Unknown),
};
```

This time, I'm not giving any number to the helper function. Let's now convert these records to Pokemons and return them:

```
...
let mut pokemons = vec![];

for record in json.records.into_iter() {
    match (
        PokemonNumber::try_from(record.fields.number),
        PokemonName::try_from(record.fields.name),
        PokemonTypes::try_from(record.fields.types),
    ) {
        (Ok(number), Ok(name), Ok(types)) => {
            pokemons.push(Pokemon::new(number, name, types))
        }
        _ => return Err(FetchAllError::Unknown),
    }
}

Ok(pokemons)
```

Already done? That's right. Let's insert Pokemons now ;)

#### Insert a Pokemon

We have an issue with Airtable: a primary key does not imply unicity. That's because they are actually using ids which are orthogonal to what we see in the tables. Maybe you already have noticed it if you've seen the `id` field in `AirtableRecord` ;) So, we can't try to insert a record and wait for an error, Airtable will nicely add the row in the table. What we are going to do then, is to first fetch Pokemons with the same number and return an error if the result of this request is not empty.

So we'll work with the following function:

```
fn insert(
    &self,
    number: PokemonNumber,
    name: PokemonName,
    types: PokemonTypes,
) -> Result<Pokemon, InsertError> {
    // code will go here
}
```

As I said before, we'll first fetch the records with the number we want to insert, and return an error if we get records:

```
let json = match self.fetch_pokemon_rows(Some(u16::from(number.clone()))) {
    Ok(json) => json,
    _ => return Err(InsertError::Unknown),
};

if !json.records.is_empty() {
    return Err(InsertError::Conflict);
}
```

Now we have done that, we are sure no record share the same number, and so we can insert our Pokemon. To do that, we'll need to create a json request. Fortunately, ureq has our back:

```
...
let body = ureq::json!({
    "records": [{
        "fields": {
            "number": u16::from(number.clone()),
            "name": String::from(name.clone()),
            "types": Vec::<String>::from(types.clone()),
        },
    }],
});
```

Finally, we still have to post the body and return the Pokemon on success:

```
...
if let Err(_) = ureq::post(&self.url)
    .set("Authorization", &self.auth_header)
    .send_json(body)
{
    return Err(InsertError::Unknown);
}

Ok(Pokemon::new(number, name, types))
```

#### Delete a Pokemon

We'll work on the following function:

```
fn delete(&self, number: PokemonNumber) -> Result<(), DeleteError> {
    // code will go here
}
```

Like in `insert`, we'll have to first get the record which share the number we get as argument. This time, we'll send back an error when the records are empty. We can't delete void, can we? ;) We'll take the first record otherwise.

```
let mut json = match self.fetch_pokemon_rows(Some(u16::from(number.clone()))) {
    Ok(json) => json,
    _ => return Err(DeleteError::Unknown),
};

if json.records.is_empty() {
    return Err(DeleteError::NotFound);
}

let record = json.records.remove(0);
```

And now, we are going to use the `id` field of the record to delete the record:

```
...
match ureq::delete(&format!("{}/{}", self.url, record.id))
    .set("Authorization", &self.auth_header)
    .call()
{
    Ok(_) => Ok(()),
    _ => Err(DeleteError::Unknown),
}
```

Aaaand, we are done with Pokemon deletion.

## Conclusion

Now you should be able to use the Pokedex with an in-memory store, an Airtable workspace, or an SQLite database. Plus you can operate it using a CLI or an HTTP API :D

That was the last article of this series. Thanks for staying until here :) I hope you'll find these articles useful. As usual, you can engage with me on Twitter, and you can be notified of the release of new articles on Twitter or by subscribing to the rss feed of this blog.

The code is accessible on [github](https://github.com/alexislozano/pokedex/tree/article-7).