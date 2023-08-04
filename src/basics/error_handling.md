# Error handling

`StdError` provides useful variants related to the `CosmWasm` smart contract development. What if 
you would like to emit errors related to your business logic?

## Custom error

We start by adding a new dependency [`thiserror`](https://docs.rs/thiserror/1.0.44/thiserror/) to
our `Cargo.toml`.

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
thiserror = "1.0.44"

[dev-dependencies]
sylvia = { version = "0.7.0", features = ["mt"] }
```

It provides an easy-to-use derive macro to set up our errors.

Let's create new file `src/error.rs`.

```rust,noplayground
use cosmwasm_std::StdError;
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("Cannot decrement count. Already at zero.")]
    CannotDecrementCount,
}
```

We annotate the `ContractError` enum with [`Error`](https://docs.rs/thiserror/1.0.44/thiserror/derive.Error.html)
macro as well as `Debug` and `PartialEq`. Variants need to be prefixed with `#[error(..)]` attribute.
First one will be called `Std` and will implement [`From`](https://doc.rust-lang.org/std/convert/trait.From.html) trait on our error. This way we can both
return standard `CosmWasm` errors and our own defined ones. For our business logic we will provide 
the `CannotDecrementCount` variant. String inside of `error(..)` attribute will provide 
[`Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html) value for readability.

Now let's add the `error` module to our project. In `src/lib.rs`:

```rust,noplayground
pub mod contract;
pub mod error;
#[cfg(test)]
pub mod multitest;
pub mod responses;
```

## Use custom error

Our error is defined. Now let's add a new `ExecMsg` variant.

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use sylvia::types::{ExecCtx, InstantiateCtx, QueryCtx};
use sylvia::{contract, entry_points};

use crate::error::ContractError;
use crate::responses::CountResponse;

pub struct CounterContract {
    pub(crate) count: Item<'static, u32>,
}

#[entry_points]
#[contract]
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

    #[msg(query)]
    pub fn count(&self, ctx: QueryCtx) -> StdResult<CountResponse> {
        let count = self.count.load(ctx.deps.storage)?;
        Ok(CountResponse { count })
    }

    #[msg(exec)]
    pub fn increment_count(&self, ctx: ExecCtx) -> StdResult<Response> {
        self.count
            .update(ctx.deps.storage, |count| -> StdResult<u32> {
                Ok(count + 1)
            })?;
        Ok(Response::default())
    }

    #[msg(exec)]
    pub fn decrement_count(&self, ctx: ExecCtx) -> Result<Response, ContractError> {
        let count = self.count.load(ctx.deps.storage)?;
        if count == 0 {
            return Err(ContractError::CannotDecrementCount);
        }
        self.count.save(ctx.deps.storage, &(count - 1))?;
        Ok(Response::default())
    }
}
```

A little to explain here. We load the count, and check if it's equal to zero. If yes, then we return
our newly defined error variant. If not, then we decrement its value.
However, this won't work. If you would try to build this you will receive:
`error[E0277]: the trait bound `cosmwasm_std::StdError: From<ContractError>` is not satisfied`.
It is because `sylvia` by default generates `dispatch` returning `Result<_, StdError>`. To 
inform `sylvia` that it should be using a new type we add `#[error(ContractError)]` attribute to 
the `contract` macro call.

```rust,noplayground
#[entry_points]
#[contract]
#[error(ContractError)]
impl CounterContract {
...
}
```

Now our contract should compile, and we are ready to test it.

## Testing

Let's create a new test expecting the proper error to be returned. In `src/multitest.rs`:

```rust,noplayground
use sylvia::multitest::App;

use crate::{contract::multitest_utils::CodeId, error::ContractError};

#[test]
fn instantiate() {
    let app = App::default();
    let code_id = CodeId::store_code(&app);

    let owner = "owner";

    let contract = code_id.instantiate(42).call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 42);

    contract.increment_count().call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 43);
}

#[test]
fn decrement_below_zero() {
    let app = App::default();
    let code_id = CodeId::store_code(&app);

    let owner = "owner";

    let contract = code_id.instantiate(1).call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 1);

    contract.decrement_count().call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 0);

    let err = contract.decrement_count().call(owner).unwrap_err();
    assert_eq!(err, ContractError::CannotDecrementCount);
}
```

We instantiate our contract with `count` equal to 1. First `decrement_count` should pass as it is
above 0. Then on the second `decrement_count` call, we will `unwrap_err` and check if it matches our 
newly defined error variant.

# Next step

In the next chapter we will talk about entry points overriding.
