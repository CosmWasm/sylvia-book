# Introducing multitest

Let me introduce the [`multitest`](https://crates.io/crates/cw-multi-test) -
library for creating tests for smart contracts in Rust.

The core idea of `multitest` is abstracting an entity of contract and
simulating the blockchain environment for testing purposes. The purpose of this
is to be able to test communication between smart contracts. It does its job
well, but it is also an excellent tool for testing single-contract scenarios.

## Update dependencies

First, we need to add ^sylvia with `mt` feature enabled to our
[`dev-dependencies`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#development-dependencies).

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
cosmwasm-std = { version = "1.3.1", features = ["staking"] }
sylvia = "0.7.0"
schemars = "0.8.12"
cosmwasm-schema = "1.3.1"
serde = "1.0.180"
cw-storage-plus = "1.1.0"

[dev-dependencies]
sylvia = { version = "0.7.0", features = ["mt"] }
```

## Creating a module for tests

Now we will create a new module, `multitest`. Let's first add it to the `src/lib.rs`

```rust,noplayground
pub mod contract;
#[cfg(test)]
pub mod multitest;
pub mod responses;
```

As this module is purely for testing purposes, we prefix it with
[`#[cfg(test)]`](https://doc.rust-lang.org/book/ch11-03-test-organization.html#the-tests-module-and-cfgtest).

Now create `src/multitest.rs`.

```rust,noplayground
use sylvia::cw_multi_test::IntoAddr;
use sylvia::multitest::App;

use crate::contract::sv::mt::{CodeId, CounterContractProxy};

#[test]
fn instantiate() {
    let app = App::default();
    let code_id = CodeId::store_code(&app);

    let owner = "owner".into_addr();

    let contract = code_id.instantiate(42).call(&owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 42);
}
```

^Sylvia generates a lot of helpers for us to make testing as easy as possible.
To simulate blockchain, we create `sylvia::multitest::App`. Then we will use it to store the code id
of our contract on the blockchain using ^sylvia generated `CodeId`.

Code id identifies our contract on the blockchain and allows us to instantiate the contract on it.
We do that using `CodeId::instantiate` method. It returns the `InstantiateProxy` type, allowing us
to set some contract parameters on a blockchain. You can inspect methods like `with_label(..)`,
`with_funds(..)` or `with_admins(..)`. Once all parameters are set you use `call` passing caller to
it as an only argument. This will return `Result<ContractProxy, ..>`. Let's `unwrap` it as it is a 
testing environment and we expect it to work correctly.

Now that we have the proxy type we have to import `CounterContractProxy` to the scope.
It's a trait defining methods for our contract.
With that setup we can call our `count` method on it. It will generate appropriate
`QueryMsg` variant underneath and send to blockchain so that we don't have to do it ourselves and
have business logic transparently shown in the test. We `unwrap` and extract the only field out of 
it and that's all.

## Next step

We tested our contract on a simulated environment. Now let's add some fluidity to it and introduce
execute messages which will alter the state of our contract.
