# The Sylvia Book

This book is about writing **Smart Contracts** using [Sylvia](https://github.com/CosmWasm/sylvia) framework.

To learn about [CosmWasm](https://github.com/CosmWasm),
on which Sylvia relies, please check [The CosmWasm Book](https://book.cosmwasm.com/index.html).

## About

This repo contains a source code for [The Sylvia Book](https://cosmwasm.github.io/sylvia-book/).

## Building

The book is built using [mdBook](https://github.com/rust-lang/mdBook).
To build it, you must first install [Rust](https://www.rust-lang.org/tools/install).

Having Rust installed, install `mdBook` using cargo:

```bash
$ cargo install mdbook
```

and then build the book from this repository:

```bash
$ mdbook build
```

To serve the book locally, run:

```bash
$ mdbook serve
```

To learn more about using `mdBook`, read its own [book](https://rust-lang.github.io/mdBook/index.html).
