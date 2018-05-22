---
title: Extractors
menu: docs_basics
weight: 170
---

# Type-safe information extraction

Actix provides facility for type-safe request information extraction. By default,
actix provides several extractor implementations.

# Path

[*Path*](../../actix-web/actix_web/struct.Path.html) provides information that can
be extracted from the Request's path. You can deserialize any variable
segment from the path.

For instance, for resource that registered for `/users/{userid}/{friend}` path
two segments could be deserialized, `userid` and `friend`. This segments 
could be extracted to a `tuple`, i.e. `Path<(u32, String)>` or structure
that implementd `Deserialize` trait from *serde* crate.

```rust
use actix_web::{App, Path, Result, http};

/// extract path info from "/users/{userid}/{friend}" url
/// {userid} -  - deserializes to a u32
/// {friend} - deserializes to a String
fn index(info: Path<(u32, String)>) -> Result<String> {
    Ok(format!("Welcome {}! {}", info.1, info.0))
}

fn main() {
    let app = App::new().resource(
        "/users/{userid}/{friend}",                    // <- define path parameters
        |r| r.method(http::Method::GET).with(index));  // <- use `with` extractor
}
```

Remember! handler function that uses extractors has to be registered with 
[*Route::with()*](../../actix-web/actix_web/dev/struct.Route.html#method.with) method.

It is also possible to extract path information to a specific type that
implements `Deserialize` trait from *serde*. Here is equivalent example that uses *serde*
instead of *tuple* type.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Path, Result, http};

#[derive(Deserialize)]
struct Info {
    userid: u32,
    friend: String,
}

/// extract path info using serde
fn index(info: Path<Info>) -> Result<String> {
     Ok(format!("Welcome {}!", info.friend))
}

fn main() {
    let app = App::new().resource(
       "/users/{userid}/{friend}",                    // <- define path parameters
       |r| r.method(http::Method::GET).with(index));  // <- use `with` extractor
}
```

# Query

Same can be done with the request's query.
[*Query*](../../actix-web/actix_web/struct.Query.html)
type provides extraction functionality. Underneath it uses *serde_urlencoded* crate.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Query, http};

#[derive(Deserialize)]
struct Info {
    username: String,
}

// this handler get called only if the request's query contains `username` field
fn index(info: Query<Info>) -> String {
    format!("Welcome {}!", info.username)
}

fn main() {
    let app = App::new().resource(
       "/index.html",
       |r| r.method(http::Method::GET).with(index)); // <- use `with` extractor
}
```

# Json

[*Json*](../../actix-web/actix_web/struct.Json.html) allows to deserialize
request body to a struct. To extract typed information from request's body,
the type `T` must implement the `Deserialize` trait from *serde*.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Json, Result, http};

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body
fn index(info: Json<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}

fn main() {
    let app = App::new().resource(
       "/index.html",
       |r| r.method(http::Method::POST).with(index));  // <- use `with` extractor
}
```

Some extractors provide a way to configure extraction process. Json extractor
[*JsonConfig*](../../actix-web/actix_web/dev/struct.JsonConfig.html) type for configuration.
When you register handler `Route::with()` returns configuration instance. In case of
*Json* extractor it returns *JsonConfig*. You can configure max size of the json
payload and custom error handler function.

Following example limits size of the payload to 4kb and uses custom error hander.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Json, HttpResponse, Result, http, error};

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// deserialize `Info` from request's body, max payload size is 4kb
fn index(info: Json<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}

fn main() {
    let app = App::new().resource(
       "/index.html", |r| {
           r.method(http::Method::POST)
              .with(index)
              .limit(4096)   // <- change json extractor configuration
              .error_handler(|err, req| {  // <- create custom error response
                  error::InternalError::from_response(
                     err, HttpResponse::Conflict().finish()).into()
              });
       });
}
```

# Form

At the moment only url-encoded forms are supported. Url encoded body
could be extracted to a specific type. This type must implement
the `Deserialize` trait from *serde* crate.

[*FormConfig*](../../actix-web/actix_web/dev/struct.FormConfig.html) allows
to configure extraction process.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Form, Result};

#[derive(Deserialize)]
struct FormData {
    username: String,
}

/// extract form data using serde
/// this handler get called only if content type is *x-www-form-urlencoded*
/// and content of the request could be deserialized to a `FormData` struct
fn index(form: Form<FormData>) -> Result<String> {
     Ok(format!("Welcome {}!", form.username))
}
# fn main() {}
```

# Multiple extractors

Actix provides extractor implementation for tuples (up to 10 elements)
which elements provide `FromRequest` impl.

For example we can use path extractor and query extractor at the same time.

```rust
#[macro_use] extern crate serde_derive;
use actix_web::{App, Query, Path, http};

#[derive(Deserialize)]
struct Info {
    username: String,
}

fn index(data: (Path<(u32, String)>, Query<Info>)) -> String {
    let (path, query) = data;
    format!("Welcome {}!", query.username)
}

fn main() {
    let app = App::new().resource(
       "/users/{userid}/{friend}",                    // <- define path parameters
       |r| r.method(http::Method::GET).with(index)); // <- use `with` extractor
}
```

# Other

Actix also provides several other extractors:

* [*State*](../../actix-web/actix_web/struct.State.html) - If you need
  access to an application state. This is similar to a `HttpRequest::state()`.
* *HttpRequest* - *HttpRequest* itself is an extractor which returns self.
  In case if you need access to the request.
* *String* - You can convert request's payload to a *String*.
  [*Example*](../../actix-web/actix_web/trait.FromRequest.html#example-1)
  is available in doc strings.
* *bytes::Bytes* - You can convert request's payload to a *Bytes*.
  [*Example*](../../actix-web/actix_web/trait.FromRequest.html#example)
  is available in doc strings.
