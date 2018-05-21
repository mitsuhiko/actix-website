---
title: What is Actix
menu: docs_intro
weight: 100
---

# A few Things

Actix is a few things.  The base of it is a powerful actor system for Rust on
top of which the `actix-web` system is built.  This is what you are most likely
going to work with.  What `actix-web` gives you is a fun and very fast web
development framework.

We call `actix-web` a small and pragmatic framework.  For all intents and purposes
it's a microframework with a few twists.  If you are already a Rust programmer
you will probably find yourself at home quickly, but even if you are coming from
another programming language you should find actix-web easy to pick up.

An application developed with `actix-web` will expose an HTTP server contained
within a native executable.  You can put this behind another HTTP server like
nginx or serve it up as such.  Even in the complete absence of another HTTP
server `actix-web` is powerful enough to provide HTTP 1 and HTTP 2 support as
well as SSL/TLS.  This makes it useful for building small services ready for
distribution.

Most importantly: `actix-web` runs on Rust 1.24 or later and it works with
stable releases.
