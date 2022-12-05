# Creating a query

We can now initialize our contract and store some data in it. Let's write `query` to read it's content.
We have already created a simple contract reacting to an empty instantiate message. Unfortunately, it
is not very useful. Let's make it a bit reactive.

## Declaring query response

Let's create a new file `src/responses.rs` which will contain responses to all the queries in our contract.

```rust,noplayground
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone, PartialEq, Eq, schemars::JsonSchema, Debug, Default)]
pub struct AdminListResp {
    pub admins: Vec<String>,
}
```

We have here similiar derives like in case of `InstantiateMsg`.
The most important ones are `Serialize` and `Deserialize` as we always want to return something
which is serializable.

`src/responses.rs` is not part of our project so let's change it. Go to `src/lib.rs` and add this module:

```rust,noplayground
pub mod contract;
pub mod responses;

use cosmwasm_std::{entry_point, DepsMut, Empty, Env, MessageInfo, Response, StdResult};

use crate::contract::{InstantiateMsg, AdminContract};

const CONTRACT: AdminContract = AdminContract::new();

#[entry_point]
pub fn instantiate(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> StdResult<Response> {
    msg.dispatch(&CONTRACT, (deps, env, info))
}
```

Now that we have response created go to your `src/contract.rs` file and declare new `query`.

```rust,noplayground
use crate::responses::AdminListResp;
use cosmwasm_std::{Addr, Deps, DepsMut, Empty, Env, MessageInfo, Order, Response, StdResult};
use cw_storage_plus::Map;
use schemars;
use sylvia::contract;

#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#}
#
#[contract]
impl AdminContract<'_> {
    ...

    #[msg(query)]
    pub fn admin_list(&self, ctx: (Deps, Env)) -> StdResult<AdminListResp> {
        let (deps, _) = ctx;

        let admins: Result<_, _> = self
            .admins
            .keys(deps.storage, None, None, Order::Ascending)
            .map(|addr| addr.map(String::from))
            .collect();

        Ok(AdminListResp { admins: admins? })
    }
}

```

With this done we can expand our `contract` macro and see that QueryMsg is generated.

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
    #[returns(AdminListResp)]
    AdminList {},
}
impl QueryMsg {
    pub fn dispatch(
        self,
        contract: &AdminContract,
        ctx: (cosmwasm_std::Deps, cosmwasm_std::Env),
    ) -> std::result::Result<sylvia::cw_std::Binary, ContractError> {
        use QueryMsg::*;
        match self {
            AdminList {} => {
                cosmwasm_std::to_binary(&contract.admin_list(ctx.into())?).map_err(Into::into)
            }
        }
    }
}
```

We will ignore `#[returns(_)]` and `cosmwasm_schema::QueryResponses` as they will be described later
when we will talk about generating schema.

`QueryMsg` is an enum which will contain every `query` declared in your expanded impl. Thanks to
that you can focus solely on defining behavior of the contract on receiving a message and you can
leave it to sylvia to generate the messages and dispatch.

Note that our enum has no type assigned to the only `AdminList` variant. Typically
in Rust, we create such variants without additional `{}` after the variant name. Here the
curly braces have a purpose - without them. The variant would serialize to just a string
type - so instead of `{ "admin_list": {} }`, the JSON representation of this variant would be
`"admin_list"`.

Instead of returning the `Response` type on the success
case, we return an arbitrary serializable object. This is because queries are not using a typical
actor model message flow - they cannot trigger any actions nor communicate with other contracts in
ways different than querying them (which is handled by the `deps` argument). The query always
returns plain data, which should be presented directly to the querier.
Sylvia does that returning encoded response to the
[`Binary`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Binary.html) by calling
[`to_binary`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/fn.to_binary.html) function in dispatch.

In case of `query` `ctx` is tuple of [`Deps`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Deps.html)
and `Env`.
We use `Deps` instead of `DepsMut` as it was done in case of instantiate is because the query can
never alter the smart contract's internal state. It can only read the state. It comes with some
consequences - for example, it is impossible to implement caching for future queries (as it would
require some data cache to write to).

The other difference is the lack of the `info` argument. The reason here is that the entry point which
performs actions (like instantiation or execution) can differ in how an action is performed based on the
message metadata - for example, they can limit who can perform an action (and do so by checking the
message `sender`). It is not a case for queries. Queries are supposed just purely to return some
transformed contract state. It can be calculated based on some chain metadata (so the state can
"automatically" change after some time), but not on message info.

Now that QueryMsg is created let's allow users to actually call it by defining entry point for
query in `src/lib.rs`.

```rust,noplayground
use cosmwasm_std::{entry_point, Deps, DepsMut, Env, MessageInfo, Response, Binary};

use crate::contract::{AdminContract, ContractQueryMsg, InstantiateMsg};

...

#[entry_point]
pub fn query(deps: Deps, env: Env, msg: ContractQueryMsg) -> Result<Binary, ContractError> {
    msg.dispatch(&CONTRACT, (deps, env))
}
```

There is one more new thing here. We were still talking about `QueryMsg`, but now we use
`ContractQueryMsg` out of nowhere. Let me explain. Sylvia framework allows us to define
[`interfaces`](https://docs.rs/sylvia/latest/sylvia/attr.interface.html). Users can create
interfaces with specific functionalities and then implement them on contract. `ContractQueryMsg` is
wrapper over `QueryMsg` from contract and it's interfaces which dispatches message to appropriate
enum. `Interfaces` will be handled further in the book.

Now, when we have the contract ready to do something, let's go and test it.
