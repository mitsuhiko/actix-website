---
title: Middlewares
menu: docs_advanced
weight: 220
---

# Middleware

Actix's middleware system allows us to add additional behavior to request/response processing.
Middleware can hook into an incoming request process, enabling us to modify requests
as well as halt request processing to return a response early.

Middleware can also hook into response processing.

Typically, middleware is involved in the following actions:

* Pre-process the Request
* Post-process a Response
* Modify application state
* Access external services (redis, logging, sessions)

Middleware is registered for each application and executed in same order as
registration. In general, a *middleware* is a type that implements the
[*Middleware trait*](../../actix-web/actix_web/middleware/trait.Middleware.html).
Each method in this trait has a default implementation. Each method can return
a result immediately or a *future* object.

The following demonstrates using middleware to add request and response headers:

```rust
use http::{header, HttpTryFrom};
use actix_web::{App, HttpRequest, HttpResponse, Result};
use actix_web::middleware::{Middleware, Started, Response};

struct Headers;  // <- Our middleware

/// Middleware implementation, middlewares are generic over application state,
/// so you can access state with `HttpRequest::state()` method.
impl<S> Middleware<S> for Headers {

    /// Method is called when request is ready. It may return
    /// future, which should resolve before next middleware get called.
    fn start(&self, req: &mut HttpRequest<S>) -> Result<Started> {
        req.headers_mut().insert(
            header::CONTENT_TYPE, header::HeaderValue::from_static("text/plain"));
        Ok(Started::Done)
    }

    /// Method is called when handler returns response,
    /// but before sending http message to peer.
    fn response(&self, req: &mut HttpRequest<S>, mut resp: HttpResponse)
        -> Result<Response>
    {
        resp.headers_mut().insert(
            header::HeaderName::try_from("X-VERSION").unwrap(),
            header::HeaderValue::from_static("0.2"));
        Ok(Response::Done(resp))
    }
}

fn main() {
    App::new()
       // Register middleware, this method can be called multiple times
       .middleware(Headers)
       .resource("/", |r| r.f(|_| HttpResponse::Ok()));
}
```

> Actix provides several useful middlewares, such as *logging*, *user sessions*, etc.

# Logging

Logging is implemented as a middleware.
It is common to register a logging middleware as the first middleware for the application.
Logging middleware must be registered for each application.

The `Logger` middleware uses the standard log crate to log information. You should enable logger
for *actix_web* package to see access log ([env_logger](https://docs.rs/env_logger/*/env_logger/)
or similar).

## Usage

Create `Logger` middleware with the specified `format`.
Default `Logger` can be created with `default` method, it uses the default format:

```ignore
  %a %t "%r" %s %b "%{Referer}i" "%{User-Agent}i" %T
```

```rust
extern crate env_logger;
use actix_web::App;
use actix_web::middleware::Logger;

fn main() {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();

    App::new()
       .middleware(Logger::default())
       .middleware(Logger::new("%a %{User-Agent}i"))
       .finish();
}
```

The following is an example of the default logging format:

```
INFO:actix_web::middleware::logger: 127.0.0.1:59934 [02/Dec/2017:00:21:43 -0800] "GET / HTTP/1.1" 302 0 "-" "curl/7.54.0" 0.000397
INFO:actix_web::middleware::logger: 127.0.0.1:59947 [02/Dec/2017:00:22:40 -0800] "GET /index.html HTTP/1.1" 200 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:57.0) Gecko/20100101 Firefox/57.0" 0.000646
```

## Format

 `%%`  The percent sign

 `%a`  Remote IP-address (IP-address of proxy if using reverse proxy)

 `%t`  Time when the request was started to process

 `%P`  The process ID of the child that serviced the request

 `%r`  First line of request

 `%s`  Response status code

 `%b`  Size of response in bytes, including HTTP headers

 `%T`  Time taken to serve the request, in seconds with floating fraction in .06f format

 `%D`  Time taken to serve the request, in milliseconds

 `%{FOO}i`  request.headers['FOO']

 `%{FOO}o`  response.headers['FOO']

 `%{FOO}e`  os.environ['FOO']

## Default headers

To set default response headers, the `DefaultHeaders` middleware can be used. The
*DefaultHeaders* middleware does not set the header if response headers already contain
a specified header.

```rust
use actix_web::{http, middleware, App, HttpResponse};

fn main() {
    let app = App::new()
        .middleware(
            middleware::DefaultHeaders::new()
                .header("X-Version", "0.2"))
        .resource("/test", |r| {
             r.method(http::Method::GET).f(|req| HttpResponse::Ok());
             r.method(http::Method::HEAD).f(|req| HttpResponse::MethodNotAllowed());
        })
       .finish();
}
```

## User sessions

Actix provides a general solution for session management. The
[**SessionStorage**](../../actix-web/actix_web/middleware/session/struct.SessionStorage.html) middleware can be
used with different backend types to store session data in different backends.

> By default, only cookie session backend is implemented. Other backend implementations
> can be added.

[**CookieSessionBackend**](../../actix-web/actix_web/middleware/session/struct.CookieSessionBackend.html)
uses cookies as session storage. `CookieSessionBackend` creates sessions which
are limited to storing fewer than 4000 bytes of data, as the payload must fit into a
single cookie. An internal server error is generated if a session contains more than 4000 bytes.

A cookie may have a security policy of *signed* or *private*. Each has a respective `CookieSessionBackend` constructor.

A *signed* cookie may be viewed but not modified by the client. A *private* cookie may neither be viewed nor modified by the client.

The constructors take a key as an argument. This is the private key for cookie session - when this value is changed, all session data is lost.

In general, you create a
`SessionStorage` middleware and initialize it with specific backend implementation,
such as a `CookieSessionBackend`. To access session data,
[*HttpRequest::session()*](../../actix-web/actix_web/middleware/session/trait.RequestSession.html#tymethod.session)
 must be used. This method returns a
[*Session*](../../actix-web/actix_web/middleware/session/struct.Session.html) object, which allows us to get or set
session data.

```rust
use actix_web::{server, App, HttpRequest, Result};
use actix_web::middleware::session::{RequestSession, SessionStorage, CookieSessionBackend};

fn index(mut req: HttpRequest) -> Result<&'static str> {
    // access session data
    if let Some(count) = req.session().get::<i32>("counter")? {
        println!("SESSION value: {}", count);
        req.session().set("counter", count+1)?;
    } else {
        req.session().set("counter", 1)?;
    }

    Ok("Welcome!")
}

fn main() {
    let sys = actix::System::new("basic-example");
    server::new(
        || App::new().middleware(
           SessionStorage::new(
             CookieSessionBackend::signed(&[0; 32])
                .secure(false)
            )))
        .bind("127.0.0.1:59880").unwrap()
        .start();
    let _ = sys.run();
}
```

# Error handlers

`ErrorHandlers` middleware allows us to provide custom handlers for responses.

You can use the `ErrorHandlers::handler()` method to register a custom error handler
for a specific status code. You can modify an existing response or create a completly new
one. The error handler can return a response immediately or return a future that resolves
into a response.

```rust
use actix_web::{
    App, HttpRequest, HttpResponse, Result,
    http, middleware::Response, middleware::ErrorHandlers};

fn render_500<S>(_: &mut HttpRequest<S>, resp: HttpResponse) -> Result<Response> {
   let mut builder = resp.into_builder();
   builder.header(http::header::CONTENT_TYPE, "application/json");
   Ok(Response::Done(builder.into()))
}

fn main() {
    let app = App::new()
        .middleware(
            ErrorHandlers::new()
                .handler(http::StatusCode::INTERNAL_SERVER_ERROR, render_500))
        .resource("/test", |r| {
             r.method(http::Method::GET).f(|_| HttpResponse::Ok());
             r.method(http::Method::HEAD).f(|_| HttpResponse::MethodNotAllowed());
        })
        .finish();
}
```
