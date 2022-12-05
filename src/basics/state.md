# Contract state

The contract we are working on can be instantiated but it doesn't do anything afterward.
Let's make it more complex. In this chapter we will introduce contracts state.

## Adding contract state

The state will be initialized on contract instantiation. The state
will contain a list of admins who would be eligible to execute messages in the future.

The first thing to do is to update `Cargo.toml` with yet another dependency - the
[`storage-plus`](https://crates.io/crates/cw-storage-plus) crate with high-level bindings for CosmWasm
smart contracts state management:

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
cw-storage-plus = "0.16.0"
```

Now add the state as a field in your contract and instantiate it in new method.

```rust,noplayground
use cosmwasm_std::{Addr, DepsMut, Empty, Env, MessageInfo, Response, StdResult};
use cw_storage_plus::Map;
use schemars;
use sylvia::contract;

pub struct AdminContract<'a> {
    pub(crate) admins: Map<'static, &'a Addr, Empty>,
}

#[contract]
impl AdminContract<'_> {
    pub const fn new() -> Self {
        Self {
            admins: Map::new("admins"),
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: (DepsMut, Env, MessageInfo),
    ) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

New types:

- [`Map<_, _>`](https://docs.rs/cw-storage-plus/0.16.0/cw_storage_plus/struct.Map.html)

- [`Addr`](https://docs.rs/cosmwasm-std/0.16.0/cosmwasm_std/struct.Addr.html) - representation of
  actual address on blockchain.

- [`Empty`](https://docs.rs/cosmwasm-std/0.16.0/cosmwasm_std/struct.Empty.html) - an empty struct
  that serves as a placeholder.

We declared state `admins` as immutable `Map<'static, &'a Addr, Empty>`.
It might seem weird that we created `Map` with `Empty` value containing no information, but our
alternative would be to store it as `Vec<Addr>` which would force us to load whole `Vec` to
alternate it or read a single element which would be a costly operation.
Because of that it is better to declare it as a `Map`.

But why isn't it mutable? How will we modify the elements?

The answer is tricky - this immutable is not keeping the state itself. The state is stored in the
blockchain and is accessed via the `deps` argument passed to entry points. The storage-plus constants
are just accessor utilities helping us access this state in a structured way.

In CosmWasm, the blockchain state is just massive key-value storage. The keys are prefixed with
metainformation pointing to the contract which owns them (so no other contract can alter them in
any way), but even after removing the prefixes, the single contract state is a smaller key-value pair.

`storage-plus` handles more complex state structures by additionally prefixing items keys intelligently.
The key to the `Map` doesn't matter to us - it would be figured out to be unique based on a unique string passed to the
[`new`](https://docs.rs/cw-storage-plus/0.13.4/cw_storage_plus/struct.Item.html#method.new) method.

## Initializing the state

Now that state field is added we can improve our instantiate. We will make it possible for user to
add new admins at contract instatiation.

```rust,noplayground
use cosmwasm_std::{Addr, DepsMut, Empty, Env, MessageInfo, Response};
use cw_storage_plus::Map;
use schemars;
use sylvia::contract;
#
#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#}
#
#[contract]
impl AdminContract<'_> {
#    pub const fn new() -> Self {
#        Self {
#            admins: Map::new("admins"),
#        }
#    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        ctx: (DepsMut, Env, MessageInfo),
        admins: Vec<String>,
    ) -> StdResult<Response> {
        let (deps, _, _) = ctx;

        for admin in admins {
            let admin = deps.api.addr_validate(&admin)?;
            self.admins.save(deps.storage, &admin, &Empty {})?;
        }

        Ok(Response::new())
    }
}
```

Voila, that's all that is needed to update the state! With this change when we expand `contract`
macro we should see:

```rust,noplayground
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
pub struct InstantiateMsg {
    pub admins: Vec<String>,
}
impl InstantiateMsg {
    pub fn dispatch(
        self,
        contract: &AdminContract<'_>,
        ctx: (
            cosmwasm_std::DepsMut,
            cosmwasm_std::Env,
            cosmwasm_std::MessageInfo,
        ),
    ) -> StdResult<Response> {
        let Self { admins } = self;
        contract.instantiate(ctx.into(), admins).map_err(Into::into)
    }
}
```

As you can see admins was set as a field of `InstantiateMsg` and in dispatch it's forwarded to
insantiate implemented in our contract.

There vector of strings is being transformed into vector of `Addr`. We cannot take addresses as a
message argument because not every string is a valid address. It might be a bit confusing when we
were working on tests. Any string could be used in the place of address.

Let me explain. Every string can be technically considered an address. However, not every string is
an actual existing blockchain address. When we keep anything of type `Addr` in the contract, we
assume it is a proper address in the blockchain. That is why the
[`addr_validate`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/trait.Api.html#tymethod.addr_validate)
function exits - to check this precondition.

Having data to store, we use the [`save`](https://docs.rs/cw-storage-plus/0.16.0/cw_storage_plus/struct.Map.html#method.save)
function to write it into the contract state. Note that the first argument of `save` is
[`&mut Storage`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/trait.Storage.html), which is
actual blockchain storage. As emphasized, the `Map` object stores nothing and is just an accessor.
It determines how to store the data in the storage given to it. The second argument is the
serializable data to be stored and the last one is the value which in our case is simply `Empty`.

Nice we now have state initialized on our contract, but we can't really validate if the data is
stored correctly. Let's change it in next chapter in which we will introduce `query`.
