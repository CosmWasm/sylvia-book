# Contract state 

We can instantiate our contract, but it doesn't do anything afterward.
Let's make it more usable. In this chapter, we will introduce the contract's state.

## Adding contract state

The name of our contract is a little spoiler. We will add the `counter` state. It's not a real world 
usage of smart contracts, but it helps to see the usage of [Sylvia](https://github.com/CosmWasm/sylvia)
without getting into business logic.

The first thing to do, is to update `Cargo.toml` with yet another dependency - the
[`storage-plus`](https://crates.io/crates/cw-storage-plus) crate with high-level bindings for
[CosmWasm](https://github.com/CosmWasm) smart contracts state management:

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
```

Now add state as a field in your contract and instantiate it in the `new` function (constructor):

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use sylvia::types::InstantiateCtx;
use sylvia::{contract, entry_points};

pub struct CounterContract {
    pub(crate) count: Item<'static, u32>,
}

#[contract]
#[entry_points]
impl CounterContract {
    pub const fn new() -> Self {
        Self {
            count: Item::new("count"),
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}
```

New type:

- [`Item<_>`](https://docs.rs/cw-storage-plus/1.1.0/cw_storage_plus/struct.Item.html) - this is 
just an **accessor** that allows to read a state stored on the blockchain via the key **"count"**
in our case. It doesn't hold any state by itself.

In [CosmWasm](https://github.com/CosmWasm), the blockchain state is just a massive key-value storage.
The keys are prefixed with metainformation, pointing to the contract which owns them
(so no other contract can alter them), but even after removing the prefixes,
the single contract state is a smaller key-value pair.

## Initializing the state

Now that the state field has been added, we can improve the instantiation of our contract.
We make it possible for a user to add an initial counter value at contract instantiation.

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use sylvia::types::InstantiateCtx;
use sylvia::{contract, entry_points};

pub struct CounterContract {
    pub(crate) count: Item<'static, u32>,
}

#[contract]
#[entry_points]
impl CounterContract {
    pub const fn new() -> Self {
        Self {
            count: Item::new("count"),
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, ctx: InstantiateCtx, count: u32) -> StdResult<Response> {
        self.count.save(ctx.deps.storage, &count)?;
        Ok(Response::default())
    }
}
```

Having data to store, we use the
[`save`](https://docs.rs/cw-storage-plus/1.1.0/cw_storage_plus/struct.Item.html#method.save)
function to write it into the contract state. Note that the first argument of `save` is
[`&mut Storage`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/trait.Storage.html), which is
actual blockchain storage. As emphasized, the `Item` object holds no state by itself,
and is just an accessor to blockchain's storage. `Item` only determines how to store
the data in the storage given to it.

Now let's expand the `contract` macro and see what changed.

```rust,noplayground
pub struct InstantiateMsg {
    pub count: u32,
}
impl InstantiateMsg {
    pub fn new(count: u32) -> Self {
        Self { count }
    }
    pub fn dispatch(
        self,
        contract: &CounterContract,
        ctx: (
            sylvia::cw_std::DepsMut,
            sylvia::cw_std::Env,
            sylvia::cw_std::MessageInfo,
        ),
    ) -> StdResult<Response> {
        let Self { count } = self;
        contract
            .instantiate(Into::into(ctx), count)
            .map_err(Into::into)
    }
}
```

First, adding parameter to the `instantiate` method added it as a field to the `InstantiateMsg`.
It also caused `dispatch` to pass this field to the `instantiate` method.
Thanks to [Sylvia](https://github.com/CosmWasm/sylvia) we don't have to tweak every function,
entry point or message, and all we need to do, is just to modify the `instantiate` function.

## Next step

Well, now we have the state initialized for our contract, but we still can't validate,
if the data we passed during instantiation is stored correctly.
Let's add it in the next chapter, in which we introduce **`query`**.
