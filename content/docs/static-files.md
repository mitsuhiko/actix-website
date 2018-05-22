---
title: Static Files
menu: docs_advanced
weight: 230
---

# Individual file

It is possible to serve static files with a custom path pattern and `NamedFile`. To
match a path tail, we can use a `[.*]` regex.

```rust
use std::path::PathBuf;
use actix_web::{App, HttpRequest, Result, http::Method, fs::NamedFile};

fn index(req: HttpRequest) -> Result<NamedFile> {
    let path: PathBuf = req.match_info().query("tail")?;
    Ok(NamedFile::open(path)?)
}

fn main() {
    App::new()
        .resource(r"/a/{tail:.*}", |r| r.method(Method::GET).f(index))
        .finish();
}
```

# Directory

To serve files from specific directories and sub-directories, `StaticFiles` can be used.
`StaticFiles` must be registered with an `App::handler()` method, otherwise
it will be unable to serve sub-paths.

```rust
use actix_web::{App, fs};

fn main() {
    App::new()
        .handler(
            "/static",
            fs::StaticFiles::new(".")
                .show_files_listing())
        .finish();
}
```

The parameter is the base directory. By default files listing for sub-directories
is disabled. Attempt to load directory listing will return *404 Not Found* response.
To enable files listing, use 
[*StaticFiles::show_files_listing()*](../../actix-web/actix_web/fs/struct.StaticFiles.html#method.show_files_listing)
method.

Instead of showing files listing for directory, it is possible to redirect
to a specific index file. Use the
[*StaticFiles::index_file()*](../../actix-web/actix_web/fs/struct.StaticFiles.html#method.index_file)
method to configure this redirect.
