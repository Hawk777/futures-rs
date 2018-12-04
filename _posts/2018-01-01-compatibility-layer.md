---
layout: post
title:  "Compatibility Layer"
subtitle: "0.1 ❤ 0.3"
author: "Josef Brandl"
author_github: "MajorBreakfast"
date:   2018-01-01
categories: blog
---

# Futures 0.1 Compatibility Layer

Rust's futures ecosystem is currently split in two: On the one hand we have the vibrant ecosystem built around [`futures@0.1`][] with its many libraries working on stable Rust and on the other hand there's the unstable [`std::future`][] ecosystem with support for the ergonomic and powerful `async`/`await` language feature. To bridge the gap between these two worlds we have introduced a compatibility layer as part of the [`futures@0.3`][] extension to `std::future`.  This blog post aims to give an overview over how to use it.

[`futures@0.1`]: https://docs.rs/futures
[`futures@0.3`]: https://rust-lang-nursery.github.io/futures-api-docs/
[`std::future`]: https://doc.rust-lang.org/nightly/std/future/

## `Cargo.toml`

The compatibility layer can be enabled by setting the `compat` feature in your `Cargo.toml`:

```toml
futures-preview = { version = "0.3.0-alpha.10", features = ["compat"] }
```

To use `futures@0.1` and `futures@0.3` together in a single project, we can make use of the [new cargo feature for renaming dependencies][renaming-dependencies]. Why? Because, even though the `futures@0.3` crate is called `futures-preview` on crates.io, it's lib name is also `futures`. By renaming `futures` version 0.1 to `futures01`, we can avoid a name collision:

[renaming-dependencies]: https://doc.rust-lang.org/nightly/cargo/reference/specifying-dependencies.html#renaming-dependencies-in-cargotoml

```toml
[dependencies]
futures01 = { package = "futures", version = "0.1", optional = true }
```

**Note: Renaming the crate is only required if you specify it as a dependency.  If your project only depends on Tokio and thus only indirectly on `futures@0.1`, then no renaming is required.**

## Async functions on 0.1 executors

The compatibility layer makes it possible to run `std` futures on executors built for `futures@0.1`. This makes it for instance possible to run futures created via `async`/`await!` on Tokio's executor. Here's how this looks like:

```rust
#![feature(async_await, await_macro, futures_api)]
use futures::future::{FutureExt, TryFutureExt};

let future03 = async {
    println!("Running on the pool");
};

let future01 = future03
    .unit_error()
    .boxed()
    .compat();

tokio::run(future01);
```

Turning a `std` future into a 0.1 future requires three steps:

- First, the future needs to be a [`TryFuture`][], i.e. a future with `Output = Result<T, E>`. If your future isn't a `TryFuture` yet, you can quickly make it one using the [`unit_error`][] combinator which wraps the output in a `Result<T, ()>`.

- Next, the future needs to be [`Unpin`][]. If your future isn't `Unpin` yet, you can use the [`boxed`][] combinator which wraps the future in a `Pin<Box>`.

- The final step is to call the [`compat`][] combinator which wraps the `std` future into a 0.1 future that can run on any 0.1 executor.

[`TryFuture`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/future/trait.TryFuture.html
[`unit_error`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/future/trait.FutureExt.html#method.unit_error
[`Unpin`]: https://doc.rust-lang.org/nightly/std/marker/trait.Unpin.html
[`boxed`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/future/trait.FutureExt.html#method.boxed
[`compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/future/trait.TryFutureExt.html#method.compat

## 0.1 futures in async functions

The conversion from a 0.1 future to a `std` future also works via a [`compat`][Future01CompatExt::compat] combinator method:

```rust
use futures::compat::Future01CompatExt;

let future03 = future01.compat();
```

It converts a 0.1 `Future<Item = T, Error = E>` into a `std` `Future<Output = Result<T, E>>`.

[Future01CompatExt::compat]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/compat/trait.Future01CompatExt.html#method.compat

## Streams and Sinks

Converting between 0.1 and 0.3 streams is possible via the [`TryStreamExt::compat`][] and [`Stream01CompatExt::compat`][] methods. Both combinators work analogously to their future equivalents.

Sinks are a little different, the current compat layer only includes support for transforming a 0.3 sink into a 0.1 sink, this goes through a [`SinkExt::compat`][] method as normal. See [rust-lang-nursery/futures-rs#1362][] for more details on the inverse transform.

[`TryStreamExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/prelude/trait.TryStreamExt.html#method.compat
[`Stream01CompatExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/compat/trait.Stream01CompatExt.html#method.compat
[`SinkExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/prelude/trait.SinkExt.html#method.compat
[rust-lang-nursery/futures-rs#1362]: https://github.com/rust-lang-nursery/futures-rs/issues/1362

## Spawning

Next, there's also conversions between futures 0.1 [`Executor`][] trait and futures 0.3 [`Spawn`][] trait. Again, these function near identically to the futures compatibility functions with [`SpawnExt::compat`][] and [`Executor01CompatExt::compat`][] providing the conversions in each direction.

[`Executor`]: https://docs.rs/futures/0.1.25/futures/future/trait.Executor.html
[`Spawn`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/task/trait.Spawn.html
[`SpawnExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/task/trait.SpawnExt.html#method.compat
[`Executor01CompatExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/compat/trait.Executor01CompatExt.html#tymethod.compat

### Tokio's Default Executor

There's an additional feature to allow easy use of Tokio's thread-local executor on libraries that are using the `Spawn` trait, adding `tokio-compat` to your `futures-preview` dependency will add an additional [`TokioDefaultSpawner`][] struct. `TokioDefaultSpawner` is an implementation of `Spawn` that will delegate any new futures to Tokio's [`DefaultExecutor`][]

[`TokioDefaultSpawner`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/compat/struct.TokioDefaultSpawner.html
[`DefaultExecutor`]: https://docs.rs/tokio/0.1.13/tokio/executor/struct.DefaultExecutor.html

## Async IO

With futures 0.1 IO was almost exclusively provided by Tokio, this was reflected in the core IO traits being provided by [`tokio-io`][]. With 0.3 instead the `futures` crate is providing these core abstractions along with generic utilities that work with them. To convert between these traits you can enable the `io-compat` feature in your `Cargo.toml` (this will implicitly enable the base `compat` feature as well):

```toml
futures-preview = { version = "0.3.0-alpha.10", features = ["io-compat"] }
```

This provides the 4 expected conversions: [`AsyncReadExt::compat`][] to convert from a [`futures::io::AsyncRead`][] to a [`tokio-io::AsyncRead`][] and [`AsyncWriteExt::compat_write`][] to convert from a [`futures::io::AsyncWrite`][] to a [`tokio-io::AsyncWrite`][], for the inverse transformations there are (currently?) no helper methods provided, but you can simply construct a [`Compat01As03`][] wrapping the reader/writer.

[`tokio-io`]: https://docs.rs/tokio-io/
[`futures::io::AsyncRead`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/io/trait.AsyncRead.html
[`futures::io::AsyncWrite`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/io/trait.AsyncWrite.html
[`tokio-io::AsyncRead`]: https://docs.rs/tokio-io/0.1.10/tokio_io/trait.AsyncRead.html
[`tokio-io::AsyncWrite`]: https://docs.rs/tokio-io/0.1.10/tokio_io/trait.AsyncWrite.html
[`AsyncReadExt::compat`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/io/trait.AsyncReadExt.html#method.compat
[`AsyncWriteExt::compat_write`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/io/trait.AsyncWriteExt.html#method.compat_write
[`Compat01As03`]: https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.10/futures/compat/struct.Compat01As03.html

## Conclusion

The compatibility layer offers conversions in both directions and thus enables gradual migrations and experiments with `std::future`. With that it manages to bridge the gap between the futures 0.1 and futures 0.3 ecosystems.

Finally a self contained example that shows how to fetch a website from a server:

```rust
#![feature(pin, async_await, await_macro)]
use futures::compat::{Compat01As03, Future01CompatExt, Stream01CompatExt};
use futures::stream::StreamExt;
use futures::future::{FutureExt, TryFutureExt};
use futures::io::AsyncWriteExt;
use hyper::Client;
use pin_utils::pin_mut;
use std::error::Error;

fn main() {
    let future03 = async {
        let url = "http://httpbin.org/ip".parse()?;

        let client = Client::new();
        let res = await!(client.get(url).compat())?;
        println!("{}", res.status());

        let body = res.into_body().compat();
        pin_mut!(body);

        let mut stdout = Compat01As03::new(tokio::io::stdout());
        while let Some(Ok(chunk)) = await!(body.next()) {
            await!(stdout.write_all(&chunk))?;
        }

        Ok(())
    };

    tokio::run(future03.map_err(|e: Box<dyn Error>| panic!("{}", e)).boxed().compat())
}
```

Special thanks goes to [@tinaun](https://www.github.com/tinaun) and [@Nemo157](https://www.github.com/Nemo157) for developing the compatibility layer.