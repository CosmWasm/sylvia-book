# Creating a query

We can now initialize our contract and store some data in it. Let's write `query` to read it's
content.

## Declaring query response

Let's create a new file, `src/responses.rs`, containing responses to all the queries in our contract.

`src/responses.rs` is not part of our project, so let's change it. Go to `src/lib.rs` and add this 
module:

```rust,noplayground
pub mod contract;
pub mod responses;
```

Now in `src/responses.rs`, we will create a response struct.

```rust,noplayground
use cosmwasm_schema::cw_serde;

#[cw_serde]
pub struct CountResponse {
    pub count: u32,
}
```

We used the [`cw_serde`](https://docs.rs/cosmwasm-schema/1.3.1/cosmwasm_schema/attr.cw_serde.html)
attribute macro here. It expands into multiple derives required by your types in the blockchain environment.

After creating a response, go to your `src/contract.rs` file and declare a new `query`.

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use sylvia::types::{InstantiateCtx, QueryCtx};
use sylvia::{contract, entry_points};

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
}
```

With this done, we can expand our `contract` macro and see that QueryMsg is generated.

```rust,noplayground
#[allow(clippy::derive_partial_eq_without_eq)]
#[derive(
    sylvia::serde::Serialize,
    sylvia::serde::Deserialize,
    Clone,
    Debug,
    PartialEq,
    sylvia::schemars::JsonSchema,
    cosmwasm_schema::QueryResponses,
)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    #[returns(CountResponse)]
    Count {},
}
impl QueryMsg {
    pub fn dispatch(
        self,
        contract: &CounterContract,
        ctx: (sylvia::cw_std::Deps, sylvia::cw_std::Env),
    ) -> std::result::Result<sylvia::cw_std::Binary, sylvia::cw_std::StdError> {
        use QueryMsg::*;
        match self {
            Count {} => {
                sylvia::cw_std::to_binary(&contract.count(Into::into(ctx))?).map_err(Into::into)
            }
        }
    }
    pub const fn messages() -> [&'static str; 1usize] {
        ["count"]
    }
    pub fn count() -> Self {
        Self::Count {}
    }
}
```

We will ignore `#[returns(_)]` and `cosmwasm_schema::QueryResponses` as they will be described later
when we will talk about generating schema.

`QueryMsg` is an enum that will contain every `query` declared in your expanded `impl`. Thanks to
that you can focus solely on defining the behavior of the contract on receiving a message, and you
can leave it to ^sylvia to generate the messages and the `dispatch`.

Note that our enum has no type assigned to the only `Count` variant. Typically
in Rust, we create variants without additional `{}` after the variant name. Here the
curly braces have a purpose. Without them, the variant would serialize to just a string
type - so instead of `{ "admin_list": {} }`, the JSON representation of this variant would be
`"admin_list"`.

Instead of returning the `Response` type on the success case, we return an arbitrary serializable 
object. It's because queries are not using a typical actor model message flow - they cannot trigger 
any actions nor communicate with other contracts in ways different than querying them (which is 
handled by the `deps` argument). The query always returns plain data, which should be presented 
directly to the querier. ^Sylvia does that by returning encoded response as
[`Binary`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/struct.Binary.html) by calling
[`to_binary`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/fn.to_binary.html) function in dispatch.

`Queries` can never alter the internal state of the smart contracts. Because of that, `QueryCtx` has
`Deps` as a field instead of `DepsMut` as it was in case of `InstantiateCtx`. It comes with some
consequences - for example, it is impossible to implement caching for future queries (as it would
require some data cache to write to).

The other difference is the lack of the `info` argument. The reason here is that the entry point 
which performs actions (like instantiation or execution) can differ in how an action is performed 
based on the message metadata - for example, they can limit who can perform an action (and do so by
checking the message `sender`). It is not a case for queries. Queries are purely to return some
transformed contract state. It can be calculated based on chain metadata (so the state can
"automatically" change after some time) but not on message info.

`#[entry_points]` generates `query` entry point as in case of `instantiate` so we don't have to do 
anything more here.

# Next step

Now, when we have the contract ready to do something, let's go and test it.
