---
title: Errors
menu: docs_advanced
weight: 180
---

# Errors

Actix uses its own [`actix_web::error::Error`][actixerror] type and
[`actix_web::error::ResponseError`][responseerror] trait for error handling
from web handlers.

If a handler returns an `Error` (referring to the [general Rust trait
`std::error::Error`][stderror]) in a `Result` that also implements the
`ResponseError` trait, actix will render that error as an HTTP response.
`ResponseError` has a single function called `error_response()` that returns
`HttpResponse`:

```rust
pub trait ResponseError: Fail {
    fn error_response(&self) -> HttpResponse {
        HttpResponse::new(StatusCode::INTERNAL_SERVER_ERROR)
    }
}
```

A `Responder` coerces compatible `Result`s into HTTP responses:

```rust
impl<T: Responder, E: Into<Error>> Responder for Result<T, E>
```

`Error` in the code above is actix's error definition, and any errors that
implement `ResponseError` can be converted to one automatically.

Actix-web provides `ResponseError` implementations for some common non-actix
errors. For example, if a handler responds with an `io::Error`, that error is
converted into an `HttpInternalServerError`:

```rust
use std::io;

fn index(req: HttpRequest) -> io::Result<fs::NamedFile> {
    Ok(fs::NamedFile::open("static/index.html")?)
}
```

See [the actix-web API documentation](responseerrorimpls) for a full list of
foreign implementations for `ResponseError`.

## An example of a custom error response

Here's an example implementation for `ResponseError`:

```rust
use actix_web::*;

#[derive(Fail, Debug)]
#[fail(display="my error")]
struct MyError {
   name: &'static str
}

// Use default implementation for `error_response()` method
impl error::ResponseError for MyError {}

fn index(req: HttpRequest) -> Result<&'static str, MyError> {
    Err(MyError{name: "test"})
}
```

`ResponseError` has a default implementation for `error_response()` that will
render a *500* (internal server error), and that's what will happen when the
`index` handler executes above.

Override `error_response()` to produce more useful results:

```rust
#[macro_use] extern crate failure;
use actix_web::{App, HttpRequest, HttpResponse, http, error};

#[derive(Fail, Debug)]
enum MyError {
   #[fail(display="internal error")]
   InternalError,
   #[fail(display="bad request")]
   BadClientData,
   #[fail(display="timeout")]
   Timeout,
}

impl error::ResponseError for MyError {
    fn error_response(&self) -> HttpResponse {
       match *self {
          MyError::InternalError => HttpResponse::new(
              http::StatusCode::INTERNAL_SERVER_ERROR),
          MyError::BadClientData => HttpResponse::new(
              http::StatusCode::BAD_REQUEST),
          MyError::Timeout => HttpResponse::new(
              http::StatusCode::GATEWAY_TIMEOUT),
       }
    }
}

fn index(req: HttpRequest) -> Result<&'static str, MyError> {
    Err(MyError::BadClientData)
}
```

# Error helpers

Actix provides a set of error helper functions that are useful for generating
specific HTTP error codes from other errors. Here we convert `MyError`, which
doesn't implement the `ResponseError` trait, to a *400* (bad request) using
`map_err`:

```rust
# extern crate actix_web;
use actix_web::*;

#[derive(Debug)]
struct MyError {
   name: &'static str
}

fn index(req: HttpRequest) -> Result<&'static str> {
    let result: Result<&'static str, MyError> = Err(MyError{name: "test"});

    Ok(result.map_err(|e| error::ErrorBadRequest(e.name))?)
}
```

See the [API documentation for actix-web's `error` module][errorhelpers] for a
full list of available error helpers.

# Compatibility with failure

Actix-web provides automatic compatibility with the [failure] library so that
errors deriving `fail` will be converted automatically to an actix error. Keep
in that those errors will render with the default *500* status code unless you
also provide your own `error_response()` implementation for them.

# Error logging

Actix logs all errors at the `WARN` log level. If an application's log level is
set to `DEBUG` and `RUST_BACKTRACE` is enabled, the backtrace is also logged.
These are configurable with environmental variables:

```
>> RUST_BACKTRACE=1 RUST_LOG=actix_web=debug cargo run
```

The `Error` type uses the cause's error backtrace if available. If the
underlying failure does not provide a backtrace, a new backtrace is constructed
pointing to the point where the conversion occurred (rather than the origin of
the error).

# Recommended practices in error handling

It might be useful to think about dividing the errors an application produces
into two broad groups: those which are intended to be be user-facing, and those
which are not.

An example of the former is that I might use failure to specify a `UserError`
enum which encapsulates a `ValidationError` to return whenever a user sends bad
input:

```rust
#[macro_use] extern crate failure;
use actix_web::{HttpResponse, http, error};

#[derive(Fail, Debug)]
enum UserError {
   #[fail(display="Validation error on field: {}", field)]
   ValidationError {
       field: String,
   }
}

impl error::ResponseError for UserError {
    fn error_response(&self) -> HttpResponse {
       match *self {
          UserError::ValidationError { .. } => HttpResponse::new(
              http::StatusCode::BAD_REQUEST),
       }
    }
}
```

This will behave exactly as intended because the error message defined with
`display` is written with the explicit intent to be read by a user.

However, sending back an error's message isn't desirable for all errors --
there are many failures that occur in a server environment where we'd probably
want the specifics to be hidden from the user. For example, if a database goes
down and client libraries start producing connect timeout errors, or if an HTML
template was improperly formatted and errors when rendered. In these cases, it
might be preferable to map the errors to a generic error suitable for user
consumption.

Here's an example that maps an internal error to a user-facing `InternalError`
with a custom message:

```rust
#[macro_use] extern crate failure;
use actix_web::{App, HttpRequest, HttpResponse, http, error, fs};

#[derive(Fail, Debug)]
enum UserError {
   #[fail(display="An internal error occurred. Please try again later.")]
   InternalError,
}

impl error::ResponseError for UserError {
    fn error_response(&self) -> HttpResponse {
       match *self {
          UserError::InternalError => HttpResponse::new(
              http::StatusCode::INTERNAL_SERVER_ERROR),
       }
    }
}

fn index(_req: HttpRequest) -> Result<&'static str, UserError> {
    fs::NamedFile::open("static/index.html").map_err(|_e| UserError::InternalError)?;
    Ok("success!")
}
```

By dividing errors into those which are user facing and those which are not, we
can ensure that we don't accidentally expose users to errors thrown by
application internals which they weren't meant to see.

[actixerror]: ../../actix-web/actix_web/error/struct.Error.html
[errorhelpers]: ../../actix-web/actix_web/error/index.html#functions
[failure]: https://github.com/rust-lang-nursery/failure
[responseerror]: ../../actix-web/actix_web/error/trait.ResponseError.html
[responseerrorimpls]: ../../actix-web/actix_web/error/trait.ResponseError.html#foreign-impls
[stderror]: https://doc.rust-lang.org/std/error/trait.Error.html
