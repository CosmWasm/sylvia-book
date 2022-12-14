# Generating first messages

We have already created a simple contract reacting to an `Empty` message on instantiate.
Unfortunately, it is not very useful. Let's make it reactive.

## Updating dependencies

First, we need to add [`sylvia`](https://crates.io/crates/sylvia) and some other crates to our project.

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
```

We had to add also some more dependencies required for generated by `sylvia` code to compile:

- [`cosmwasm-schema`](https://docs.rs/cosmwasm-schema/latest/cosmwasm_schema/) - we will later rely
  on this dependency to generate the API schema of our contract.
- [`serde`](https://docs.rs/serde/latest/serde/) - very important framework for serializing and
  deserializing Rust data structures. It is crucial as serialization is required for messages to be
  sent from and to the `contract`.
- [`schemars`](https://docs.rs/schemars/latest/schemars/) - it will also be required later when
  dealing with generating the schema of our API.

## Creating an instantiation message

For, this step we will create a new file:

- `src/contract.rs` - here, we will define our messages and behavior of the contract upon receiving
  them

Add this module to `src/lib.rs`. You want it to be public, as users might want to get access to
types stored inside your contract.

```rust,noplayground
pub mod contract;

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
```

Now let's create an `instantiate` method for our contract. In `src/contract.rs`

```rust,noplayground
use cosmwasm_std::{DepsMut, Env, MessageInfo, Response, StdResult};
use schemars;
use sylvia::contract;

pub struct AdminContract;

pub type ContractError = cosmwasm_std::StdError;

#[contract]
impl AdminContract {
    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: (DepsMut, Env, MessageInfo)) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

So what is going on here? First, we define the Admin struct. It is empty right now but later when we
will learn about states and their fields will be used to store them.
We introduce the alias `ContractError`, required by `sylvia`. Later we will create our custom
`ContractError`, but for now, it is enough to just alias the
[`cosmwasm_std::StdError`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/enum.StdError.html).
Notice it is marked as `pub` as we want it to be accessible in other modules.
Next, we create an `impl` block for `Admin` and invoke the `contract `attribute macro from `sylvia`.
It will generate a lot of boilerplate for us regarding messages generation and their dispatch. It
will also make sure that none of the messages overlap and will catch it on compile time.
Then there is a method instantiate which currently doesn't do much.
What's important here is the `#[msg(instantiate)]`. `contract` macro will parse through the method
and, based on its structure, generate InstantiateMsg with parameters equal to ones declared in
`instantiate` method past `_ctx`. F.e., this will generate among others

```rust,noplayground
impl AdminContract {
    pub fn instantiate(
        &self,
        _ctx: (DepsMut, Env, MessageInfo),
    ) -> StdResult<Response> {
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
        contract: &AdminContract,
        ctx: (
            cosmwasm_std::DepsMut,
            cosmwasm_std::Env,
            cosmwasm_std::MessageInfo,
        ),
    ) -> StdResult<Response> {
        let Self {} = self;
        contract.instantiate(ctx.into()).map_err(Into::into)
    }
}
```

Let's focus on instantiate right now. As you can see, struct InstantiateMsg with all needed derives
is being generated for you. Most important are
[`Serialize`](https://docs.rs/serde/latest/serde/trait.Serialize.html)
and [`Deserialize`](https://docs.rs/serde/latest/serde/trait.Deserialize.html)
There is also `#[serde(rename_all = "snake_case")]`, which is not important for now.

Sylvia also generates a `dispatch` method for every msg linking it with user specified-behavior. One
of its arguments is `AdminContract`, on which the message should be dispatched. Let's create the
`new` method:

`src/contract.rs`

```rust,noplayground
use cosmwasm_std::{DepsMut, Env, MessageInfo, Response, StdResult};
use schemars;
use sylvia::contract;

pub struct AdminContract;

pub type ContractError = cosmwasm_std::StdError;

#[contract]
impl AdminContract {
    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: (DepsMut, Env, MessageInfo)) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

Great, now we can modify the `src/lib.rs`

```rust,noplayground
pub mod contract;

use cosmwasm_std::{entry_point, DepsMut, Empty, Env, MessageInfo, Response, StdResult};

use crate::contract::{InstantiateMsg, AdminContract};

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    msg.dispatch(&AdminContract, (deps, env, info))
}
```

`Empty` is now changed to `InstantiateMsg` generated by `sylvia`.
We dispatch the message, and it will trigger `instatiation` method.
For now, it is just returning an empty Response, but let's change it in the following chapters,
where we will introduce states and queries.
