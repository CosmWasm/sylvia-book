# Generating first messages

We have already created a simple contract reacting to an `Empty` message on instantiate. Unfortunately, it
is not very useful. Let's make it a bit reactive.

## Updating dependencies

First we need to add [`sylvia`](https://crates.io/crates/sylvia) and some other crates to our project.

```toml
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
cosmwasm-std = { version = "1.1", features = ["staking"] }
cosmwasm-schema = "1.1.6"
serde = { version = "1.0.147", features = ["derive"] }
sylvia = "0.2.1"
schemars = "0.8.11"
thiserror = "1.0.37"
```

## Creating instantiation message

For this step we will create two files:
- `src/error.rs` - here we will define our custom error messages
- `src/contract.rs` - here we will define our messages and behavior of the contract upon receiving them

Add these modules to `src/lib.rs`. You want them to be public as users might want to get access to types stored inside your contract.

```rust,noplayground
pub mod contract;
pub mod error;

#[cfg(not(feature = "library"))]
mod entry_points {
    use cosmwasm_std::{entry_point, DepsMut, Empty, Env, MessageInfo, Response, StdResult};

    #[entry_point]
    pub fn instantiate(
        _deps: DepsMut,
        _env: Env,
        _info: MessageInfo,
        _msg: Empty,
    ) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

Now in `src/error.rs` we will define simple enum. Currently there is not much going on in our contract so it is enough for it to forward `StdError`.

```rust,noplayground
use cosmwasm_std::StdError;
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),
}
```

With this defined we can create our InstantiateMsg. In `src/contract.rs`

```rust,noplayground
use crate::error::ContractError;
use cosmwasm_std::{DepsMut, Env, MessageInfo, Response};
use schemars;
use sylvia::contract;

pub struct Admin {}

#[contract]
impl Admin {
    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, ContractError> {
        Ok(Response::new())
    }
}
```

So what is going on here? First we define Admin struct. It is empty right now but later when we will learn about states and its fields will be used to store them.
Next we create `impl` block for `Admin` and invoke `contract `attribute macro from `sylvia`. It will generate a lot of boilerplate for us regarding messages generation and their dispatch. It will also make sure that none of the messages overlap and will catch it on compile time.
Then there is a method instantiate which currently doesn't do much.
What's important here is the `#[msg(instantiate)]`. `contract` macro will parse through the method and basing on it's structure generate InstantiateMsg with parameters equal to ones declared in `instantiate` method past `_ctx`. F.e. this will generate among others 

```rust,noplayground
impl Admin {
    pub const fn new() -> Self {
        Self {}
    }

    pub fn instantiate(
        &self,
        _ctx: (DepsMut, Env, MessageInfo),
    ) -> Result<Response, ContractError> {
        Ok(Response::new())
    }
}
#[allow(clippy::derive_partial_eq_without_eq)]
#[derive(
    sylvia::serde::Serialize,
    sylvia::serde::Deserialize,
    Clone,
    Debug,
    PartialEq,
    sylvia::schemars::JsonSchema,
)]
#[serde(rename_all = "snake_case")]
pub struct InstantiateMsg {}

impl InstantiateMsg {
    pub fn dispatch(
        self,
        contract: &Admin,
        ctx: (
            cosmwasm_std::DepsMut,
            cosmwasm_std::Env,
            cosmwasm_std::MessageInfo,
        ),
    ) -> Result<Response, ContractError> {
        let Self {} = self;
        contract.instantiate(ctx.into()).map_err(Into::into)
    }
}
```

Let's focus on instantiate right now. As you can see struct InstantiateMsg with all needed derives is being generated for you.
Most important are [`Serialize`](https://docs.rs/serde/latest/serde/trait.Serialize.html) and [`Deserialize`](https://docs.rs/serde/latest/serde/trait.Deserialize.html)
There is also `#[serde(rename_all = "snake_case")]`. It's purpose is for message to be generated in schema.json as insantiate_msg as it is a standarized way.

Sylvia also generates dispatch method for every msg linking it with user specifed behavior. Let's use the dispatch method in our entry point to instantiate the contract.

`src/lib.rs`

```rust,noplayground
pub mod contract;
pub mod error;

#[cfg(not(feature = "library"))]
mod entry_points {
    use cosmwasm_std::{entry_point, DepsMut, Empty, Env, MessageInfo, Response, StdResult};

    use crate::contract::{InstantiateMsg, AdminContract};
    use crate::error::ContractError;
    
    const CONTRACT: AdminContract = AdminContract::new();

    #[entry_point]
    pub fn instantiate(
        deps: DepsMut,
        env: Env,
        info: MessageInfo,
        msg: InstantiateMsg,
    ) -> Result<Response, ContractError> {
        msg.dispatch(&CONTRACT, (deps, env, info))
    }
}
```

We have created `const CONTRACT` on which the instantiation will be called.
Empty message is now changed to InstantiateMsg generated by sylvia on which we call dispatch which will call implementation defined in contract.rs.
For now it is just returning empty Response, but let's change it in next chapters introducing states and queries.
