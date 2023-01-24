# Create a Rust project

As smart contracts are Rust library crates, we will start with creating one:

```
cargo new --lib ./contract
```

You created a simple Rust library, but it is not yet ready to be a smart contract. The first thing
to do is to update the `Cargo.toml` file:

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
cosmwasm-std = { version = "1.1", features = ["staking"] }

```

As you can see, I added a `crate-type` field for the library section. Generating the
[`cdylib`](https://doc.rust-lang.org/reference/linkage.html) is
required to create a proper web assembly binary. The downside of this is that such a library cannot
be used as a dependency for other Rust crates - for now, it is not needed, but later we will show
how to approach reusing contracts as dependencies.

Additionally, we added one core dependency for smart contracts: the
[`cosmwasm-std`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/). This crate is a
standard library for smart contracts. It provides essential utilities for communication with the
outside world, helper functions, and types. Every smart contract we will build will
use this dependency.
