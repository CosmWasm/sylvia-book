# Custom messages

*Since version 0.8.0.*

Blockchain creators might define chain-specific logic triggered through defined by them messages.
`CosmWasm` provides a way to send such messages through `cosmwasm_std::CosmosMsg::Custom(..)` variant.

^Sylvia supports this feature, allowing both custom interfaces and contracts.

## Custom interface 

To make further code examples simpler to read, let's consider we have the custom query and exec defined as such:

`src/messages.rs`
```rust
use cosmwasm_schema::cw_serde;
use cosmwasm_std::{CustomMsg, CustomQuery};

#[cw_serde]
pub enum ExternalMsg {
    Poke,
}

#[cw_serde]
pub enum ExternalQuery {
    IsPoked,
}

impl CustomQuery for ExternalQuery {}
impl CustomMsg for ExternalMsg {}
```

Notice that query has to implement `cosmwasm_std::CustomQuery`, and the message has to implement `cosmwasm_std::CustomMsg`.

Now that we have our messages defined, we should consider if our new interface is meant to work only on a specific chain
or if the developer implementing it should be free to choose which chain it should support.

To enforce support for the specific chain, the user has to use `#[sv::custom(msg=.., query=..)]` attribute.
Once `msg` or `query` is defined, it will be enforced that all messages or queries used in the interface
will use them. This is necessary for `dispatch` to work.

`src/sv_custom.rs`
```rust
use crate::messages::{ExternalMsg, ExternalQuery};
use cosmwasm_std::{Response, StdError};
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

#[interface]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
pub trait SvCustom {
    type Error: From<StdError>;

    #[msg(exec)]
    fn sv_custom_exec(
        &self,
        ctx: ExecCtx<ExternalQuery>,
    ) -> Result<Response<ExternalMsg>, Self::Error>;

    #[msg(query)]
    fn sv_custom_query(&self, ctx: QueryCtx<ExternalQuery>) -> Result<String, Self::Error>;
}
```

If, however, we would like to give the developers the freedom to choose which chain to support, we can use define
the interface with the associated type instead.

`src/associated.rs`
```rust
use cosmwasm_std::{CustomMsg, CustomQuery, Response, StdError};
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

#[interface]
pub trait Associated {
    type Error: From<StdError>;
    type ExecC: CustomMsg;
    type QueryC: CustomQuery;

    #[msg(exec)]
    fn associated_exec(&self, ctx: ExecCtx<Self::QueryC>)
        -> Result<Response<Self::ExecC>, Self::Error>;

    #[msg(query)]
    fn associated_query(&self, ctx: QueryCtx<Self::QueryC>) -> Result<String, Self::Error>;
}
```

`ExecC` and `QueryC` associated types are reserved for custom messages and queries, respectively, and should not
be used in other contexts.

## Custom contract

Before we implement the interfaces, let's create the contract that will use them.
In the case of a contract, there is only one way to define the custom `msg` and `query`. We do it through
`#[sv::custom(..)]` attribute.

`src/contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use sylvia::{contract, types::InstantiateCtx};

use crate::messages::{ExternalMsg, ExternalQuery};

pub struct CustomContract;

#[contract]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
impl CustomContract {
    pub const fn new() -> Self {
        Self
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<ExternalQuery>,
    ) -> StdResult<Response<ExternalMsg>> {
        Ok(Response::new())
    }
}
```

As in the case of interfaces, remember to make `Deps` and `Response` generic over custom `msg` and `query`, respectively.

## Implement custom interface on contract

Now that we have defined the interfaces, we can implement them on the contract. Because it would be impossible to
cast `Response<Custom>` to `Response<Empty>` or `Deps<Empty>` to `Deps<Custom>`, implementation of the custom interface
on non-custom contracts is not possible. It is possible, however, to implement a non-custom interface on a custom contract.

Implementation of chain-specific custom interfaces is simple. We have to pass the `sv::custom(..)` attribute once again.

`src/sv_custom.rs`
```rust
use super::SvCustom;
use crate::contract::CustomContract;
use crate::messages::{ExternalMsg, ExternalQuery};
use cosmwasm_std::{Response, StdError};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

#[contract(module=crate::contract)]
#[messages(crate::sv_custom as SvCustom)]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
impl SvCustom for CustomContract {
    type Error = StdError;

    #[msg(exec)]
    fn sv_custom_exec(
        &self,
        _ctx: ExecCtx<ExternalQuery>,
    ) -> Result<Response<ExternalMsg>, Self::Error> {
        Ok(Response::new())
    }

    #[msg(query)]
    fn sv_custom_query(&self, _ctx: QueryCtx<ExternalQuery>) -> Result<String, Self::Error> {
        Ok(String::default())
    }
}
```

To implement the interface with the associated type, we have to assign types for them.
Because the type of `ExecC` and `QueryC` is defined by the user, the interface is reusable in the context of
different chains.

`src/associated.rs`
```rust
use super::Associated;
use crate::contract::CustomContract;
use crate::messages::{ExternalMsg, ExternalQuery};
use cosmwasm_std::{Response, StdError};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

#[contract(module=crate::contract)]
#[messages(crate::associated as Associated)]
impl Associated for CustomContract {
    type Error = StdError;
    type ExecC = ExternalMsg;
    type QueryC = ExternalQuery;

    #[msg(exec)]
    fn associated_exec(
        &self,
        _ctx: ExecCtx<Self::QueryC>,
    ) -> Result<Response<Self::ExecC>, Self::Error> {
        Ok(Response::new())
    }

    #[msg(query)]
    fn associated_query(&self, _ctx: QueryCtx<Self::QueryC>) -> Result<String, Self::Error> {
        Ok(String::default())
    }
}
```

## `messages` attribute on main `contract` call

We implemented the custom interfaces on the contract. Now, we can use it in the contract itself.
Because the contract is already defined as `custom,` we only need to use the `messages` attribute as in the case of
non-custom interfaces.

`src/contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use sylvia::{contract, types::InstantiateCtx};

use crate::messages::{ExternalMsg, ExternalQuery};

pub struct CustomContract;

#[contract]
#[messages(crate::sv_custom as SvCustomInterface)]
#[messages(crate::associated as AssociatedInterface)]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
impl CustomContract {
    pub const fn new() -> Self {
        Self
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<ExternalQuery>,
    ) -> StdResult<Response<ExternalMsg>> {
        Ok(Response::new())
    }
}
```

Nice and easy, isn't it?
Only one thing that needs to be added. I mentioned earlier that it is possible to implement a non-custom interface on a custom contract.
Let's define a non-custom interface.

`src/non_custom.rs`
```rust
use cosmwasm_std::{Response, StdError};
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

#[interface]
pub trait NonCustom {
    type Error: From<StdError>;

    #[msg(exec)]
    fn non_custom_exec(&self, ctx: ExecCtx) -> Result<Response, Self::Error>;

    #[msg(query)]
    fn non_custom_query(&self, ctx: QueryCtx) -> Result<String, Self::Error>;
}

pub mod impl_non_custom {
    use crate::contract::CustomContract;
    use cosmwasm_std::{Response, StdError};
    use sylvia::contract;
    use sylvia::types::{ExecCtx, QueryCtx};

    use super::NonCustom;

    #[contract(module=crate::contract)]
    #[messages(crate::non_custom as NonCustom)]
    #[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
    impl NonCustom for CustomContract {
        type Error = StdError;

        #[msg(exec)]
        fn non_custom_exec(&self, _ctx: ExecCtx) -> Result<Response, Self::Error> {
            Ok(Response::new())
        }

        #[msg(query)]
        fn non_custom_query(&self, _ctx: QueryCtx) -> Result<String, Self::Error> {
            Ok(String::default())
        }
    }
}
```

As you can see, although it's non-custom, we still have to inform ^sylvia custom types from the contract.
It's required for the `MultiTest` helpers to be generic over the same types as the contract.

Let's add the last `messages` attribute to the contract. It has to end with `: custom(msg query)`. This way ^sylvia
will know that it has to cast `Response<Custom>` to `Response<Empty>` for `msg` and `Deps<Custom>` to `Deps<Empty>` for `query`.

`src/contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use sylvia::{contract, types::InstantiateCtx};

use crate::messages::{ExternalMsg, ExternalQuery};

pub struct CustomContract;

#[contract]
#[messages(crate::sv_custom as SvCustomInterface)]
#[messages(crate::associated as AssociatedInterface)]
#[messages(crate::non_custom as NonCustom: custom(msg query))]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
impl CustomContract {
    pub const fn new() -> Self {
        Self
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<ExternalQuery>,
    ) -> StdResult<Response<ExternalMsg>> {
        Ok(Response::new())
    }
}
```

## Test custom contract

Contract and interfaces implemented. We can finally test it. 

Before setting up the test environment, we have to add to our contract logic
sending the custom messages. Let's add `query` and `exec` messages to our contract.

`src/contract.rs`
```rust
use crate::messages::{ExternalMsg, ExternalQuery};
use cosmwasm_std::{CosmosMsg, QueryRequest, Response, StdResult};
use sylvia::contract;
use sylvia::types::{ExecCtx, InstantiateCtx, QueryCtx};

#[cfg(not(feature = "library"))]
use sylvia::entry_points;

pub struct CustomContract;

#[cfg_attr(not(feature = "library"), entry_points)]
#[contract]
#[messages(crate::sv_custom as SvCustomInterface)]
#[messages(crate::associated as AssociatedInterface)]
#[messages(crate::non_custom as NonCustom: custom(msg query))]
#[sv::custom(msg=ExternalMsg, query=ExternalQuery)]
impl CustomContract {
    pub const fn new() -> Self {
        Self
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<ExternalQuery>,
    ) -> StdResult<Response<ExternalMsg>> {
        Ok(Response::new())
    }

    #[msg(exec)]
    pub fn poke(&self, _ctx: ExecCtx<ExternalQuery>) -> StdResult<Response<ExternalMsg>> {
        let msg = CosmosMsg::Custom(ExternalMsg::Poke {});
        let resp = Response::default().add_message(msg);
        Ok(resp)
    }

    #[msg(query)]
    pub fn is_poked(&self, ctx: QueryCtx<ExternalQuery>) -> StdResult<bool> {
        let resp = ctx
            .deps
            .querier
            .query::<bool>(&QueryRequest::Custom(ExternalQuery::IsPoked {}))?;
        Ok(resp)
    }
}
```

Message `poke` will return the `Response` with the `CosmosMsg::Custom(ExternalMsg::Poke {})` message.
In `is_poked` we will trigger the custom module by sending [`QueryRequest::Custom`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/enum.QueryRequest.html) via [`cosmwasm_std::QuerierWrapper::query`](https://docs.rs/cosmwasm-std/latest/cosmwasm_std/struct.QuerierWrapper.html#method.query).

The contract is able to send our custom messages. We can start creating a test environment.
First, we have to define a custom module that will handle our custom messages.
It will be simple module tracking if it has received the `ExternalMsg::Poke` message.
To add handling of custom messages, we have to implement [`cw_multi_test::Module`](https://docs.rs/cw-multi-test/latest/cw_multi_test/trait.Module.html) on it.
We are only interested in `execute` and `query` methods in our example. In case of `sudo` we will return `Ok(..)` response.

`src/multitest/custom_module.rs`
```rust
use cosmwasm_schema::schemars::JsonSchema;
use cosmwasm_std::{
    to_binary, Addr, Api, Binary, BlockInfo, CustomQuery, Empty, Querier, StdResult, Storage,
};
use cw_multi_test::{AppResponse, CosmosRouter, Module};
use cw_storage_plus::Item;
use serde::de::DeserializeOwned;
use std::fmt::Debug;

use crate::messages::{ExternalMsg, ExternalQuery};

pub struct CustomModule {
    pub is_poked: Item<'static, bool>,
}

impl CustomModule {
    pub fn new() -> Self {
        Self {
            is_poked: Item::new("is_poked"),
        }
    }

    pub fn setup(&self, storage: &mut dyn Storage) -> StdResult<()> {
        self.is_poked.save(storage, &true)
    }
}

impl Module for CustomModule {
    type ExecT = ExternalMsg;
    type QueryT = ExternalQuery;
    type SudoT = Empty;

    fn execute<ExecC, QueryC>(
        &self,
        _api: &dyn Api,
        storage: &mut dyn Storage,
        _router: &dyn CosmosRouter<ExecC = ExecC, QueryC = QueryC>,
        _block: &BlockInfo,
        _sender: Addr,
        msg: Self::ExecT,
    ) -> anyhow::Result<AppResponse>
    where
        ExecC: Debug + Clone + PartialEq + JsonSchema + DeserializeOwned + 'static,
        QueryC: CustomQuery + DeserializeOwned + 'static,
    {
        match msg {
            ExternalMsg::Poke {} => {
                self.is_poked.save(storage, &true)?;
                Ok(AppResponse::default())
            }
        }
    }

    fn sudo<ExecC, QueryC>(
        &self,
        _api: &dyn Api,
        _storage: &mut dyn Storage,
        _router: &dyn CosmosRouter<ExecC = ExecC, QueryC = QueryC>,
        _block: &BlockInfo,
        _msg: Self::SudoT,
    ) -> anyhow::Result<AppResponse>
    where
        ExecC: Debug + Clone + PartialEq + JsonSchema + DeserializeOwned + 'static,
        QueryC: CustomQuery + DeserializeOwned + 'static,
    {
        Ok(AppResponse::default())
    }

    fn query(
        &self,
        _api: &dyn Api,
        storage: &dyn Storage,
        _querier: &dyn Querier,
        _block: &BlockInfo,
        request: Self::QueryT,
    ) -> anyhow::Result<Binary> {
        match request {
            ExternalQuery::IsPoked {} => {
                let is_poked = self.is_poked.load(storage)?;
                to_binary(&is_poked).map_err(Into::into)
            }
        }
    }
}
```

With this module, we can move to testing.
We are going to use [`BasicAppBuilder`](https://docs.rs/cw-multi-test/latest/cw_multi_test/type.BasicAppBuilder.html) to save ourselves from typing the all the generics.
It's fine as we are only interested in setting `ExecC` and `QueryC` types.
Installing our module is done via [`with_custom`](https://docs.rs/cw-multi-test/latest/cw_multi_test/type.BasicAppBuilder.html) method.
In case you need to initialize some values in your module, you can use [`build`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.AppBuilder.html#method.build) method.
Generics are set on `mt_app`, and there is no need to set them on `App`.
Running `poke` on our contract will send the `ExternalMsg::Poke`, which `App` will dispatch to our custom module.

`src/multitest/tests.rs`
```rust
use sylvia::multitest::App;

use crate::contract::mt::CodeId;
use crate::multitest::custom_module::CustomModule;

#[test]
fn test_custom() {
    let owner = "owner";

    let mt_app = cw_multi_test::BasicAppBuilder::new_custom()
        .with_custom(CustomModule::new())
        .build(|router, _, storage| {
            router.custom.setup(storage).unwrap();
        });

    let app = App::new(mt_app);

    let code_id = CodeId::store_code(&app);

    let contract = code_id.instantiate().call(owner).unwrap();

    contract.poke().call(owner).unwrap();

    let count = contract.is_poked().unwrap();
    assert!(count);
}
```

# Next step

We now know how to trigger chain-specific functionality. In the next chapter, we will expand a little on that by
exploring support for generics.
