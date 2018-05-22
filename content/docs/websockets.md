---
title: Websockets
menu: docs_proto
weight: 240
---

Actix supports WebSockets out-of-the-box. It is possible to convert a request's `Payload`
to a stream of [*ws::Message*](../../actix-web/actix_web/ws/enum.Message.html) with
a [*ws::WsStream*](../../actix-web/actix_web/ws/struct.WsStream.html) and then use stream
combinators to handle actual messages, but it is simpler to handle websocket communications
with an http actor.

The following is an example of a simple websocket echo server:

```rust
use actix::*;
use actix_web::*;

/// Define http actor
struct Ws;

impl Actor for Ws {
    type Context = ws::WebsocketContext<Self>;
}

/// Handler for ws::Message message
impl StreamHandler<ws::Message, ws::ProtocolError> for Ws {

    fn handle(&mut self, msg: ws::Message, ctx: &mut Self::Context) {
        match msg {
            ws::Message::Ping(msg) => ctx.pong(&msg),
            ws::Message::Text(text) => ctx.text(text),
            ws::Message::Binary(bin) => ctx.binary(bin),
            _ => (),
        }
    }
}

fn main() {
    App::new()
      .resource("/ws/", |r| r.f(|req| ws::start(req, Ws)))
      .finish();
}
```

> A simple websocket echo server example is available in the
> [examples directory](https://github.com/actix/examples/tree/master/websocket/).

> An example chat server with the ability to chat over a websocket or tcp connection
> is available in [websocket-chat directory](https://github.com/actix/examples/tree/master/websocket-chat/)
