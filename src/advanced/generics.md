# Generics

When implementing a contract, users might want to define some generic data it should store.
One might even want the message to have some generic parameters or return type.
^Sylvia supports generics in the contracts and interfaces.

## Prepare project

To improve readability and focus solely on the feature support in this chapter, paste this
dependencies to your project `Cargo.toml`.
It contains every dependency we will use in this chapter.

```toml
[package]
name = "generic"
version = "0.1.0"
edition = "2021"

[dependencies]
sylvia = "0.9.0"
cosmwasm-std = "1.5"
schemars = "0.8"
serde = "1"
cosmwasm-schema = "1.5"
cw-storage-plus = "1.1.0"

[dev-dependencies]
anyhow = "1.0"
cw-multi-test = "0.16"
sylvia = { version = "0.9.0", features = ["mt"] }
```

## Generic interface

To define a generic interface, we don't have to add any additional attributes. Simply add
generics to the trait as you would in regular rust code.

`src/generic.rs`
```rust
use cosmwasm_std::{CustomMsg, Response, StdError, StdResult};
use serde::de::DeserializeOwned;
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

#[interface]
pub trait Generic<ExecParam, QueryParam>
where
    ExecParam: CustomMsg + DeserializeOwned,
    QueryParam: CustomMsg + DeserializeOwned,
{
    type Error: From<StdError>;

    #[msg(exec)]
    fn generic_exec(&self, ctx: ExecCtx, param: ExecParam) -> StdResult<Response>;

    #[msg(query)]
    fn generic_query(&self, ctx: QueryCtx, param: QueryParam) -> StdResult<String>;
}
```

Simple and expected behavior.
What if we want to extend the `custom` functionality with generics?
In such a case, we would have to add `#[sv::custom(..)]` attribute to the trait with the name of our generics.
It is a standard approach when using `CustomMsg` and `CustomQuery` in ^sylvia so there is nothing new here.

`src/generic.rs`
```rust
use cosmwasm_std::{CustomMsg, CustomQuery, Response, StdError, StdResult};
use serde::de::DeserializeOwned;
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

#[interface]
#[sv::custom(msg = MsgCustom, query = QueryCustom)]
pub trait Generic<ExecParam, QueryParam, QueryCustom, MsgCustom>
where
    ExecParam: CustomMsg + DeserializeOwned,
    QueryParam: CustomMsg + DeserializeOwned,
    QueryCustom: CustomQuery,
    MsgCustom: CustomMsg + DeserializeOwned,
{
    type Error: From<StdError>;

    #[msg(exec)]
    fn generic_exec(
        &self,
        ctx: ExecCtx<QueryCustom>,
        param: ExecParam,
    ) -> StdResult<Response<MsgCustom>>;

    #[msg(query)]
    fn generic_query(
        &self,
        ctx: QueryCtx<QueryCustom>,
        param: QueryParam,
    ) -> StdResult<String>;
}
```

## Implement a generic interface on the contract

Defining generic interfaces is as simple as defining generic traits.
Implementing one on the contract is also straightforward.

We will start by defining the types we will use in place of generics.

`src/messages.rs`
```rust
use cosmwasm_schema::cw_serde;
use cosmwasm_std::{CustomMsg, CustomQuery};

#[cw_serde]
pub struct MyMsg;

impl CustomMsg for MyMsg {}

#[cw_serde]
pub struct MyQuery;

impl CustomQuery for MyQuery {}
```

We also need a contract on which we will implement the interface.

`src/contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct NonGenericContract;

#[contract]
impl NonGenericContract {
    pub fn new() -> Self {
        Self {}
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

We are set and ready to implement the interface.

`src/generic_impl.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

use crate::contract::NonGenericContract;
use crate::generic::Generic;
use crate::messages::{MyMsg, MyQuery};

#[contract(module=crate::contract)]
#[messages(crate::generic as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl Generic<MyMsg, MyMsg, MyQuery, MyMsg> for NonGenericContract {
    type Error = StdError;

    #[msg(exec)]
    fn generic_exec(&self, _ctx: ExecCtx<MyQuery>, _param: MyMsg) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }

    #[msg(query)]
    fn generic_query(&self, _ctx: QueryCtx<MyQuery>, _param: MyMsg) -> StdResult<String> {
        Ok(String::default())
    }
}
```

To implement a generic interface, we have to specify the types. Here we use `MyMsg` and `MyQuery`
we defined earlier. No additional attributes have to be passed as ^sylvia will read them
from the interface type.
Because we defined the interface as `custom` we have to pass these types into `sv::custom(..)` attribute
same as in the interface definition.

## Update impl contract

Now that we have implemented the generic interface on our contract, we can inform
main `sylvia::contract` call about it.

`src/contract.rs`
```rust
use crate::messages::{MyMsg, MyQuery};
use cosmwasm_std::{Response, StdResult};
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct NonGenericContract;

#[contract]
#[messages(crate::generic<MyMsg, MyMsg, MyQuery, MyMsg> as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl NonGenericContract {
    pub fn new() -> Self {
        Self {}
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx<MyQuery>) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }
}
```

First of all, we added types to the `#[messages(..)]` attribute.
We have to add them in the same order as in the interface definition.
We also had to add `#[sv::custom(..)]` attribute to the contract definition and update 
`InstantiateCtx` and `Response` types as our interface is `custom`. 

## Generic contract

Using generic interfaces in ^sylvia is simple. Let's see how it works with generic contracts.

Let us define a new module in which we will define a generic contract.

`src/generic_contract.rs`
```rust
use cosmwasm_std::{CustomMsg, Response, StdResult};
use serde::de::DeserializeOwned;
use std::marker::PhantomData;
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<'static, DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[contract]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + DeserializeOwned + 'static,
    for<'data> DataType: 'data,
{
    pub fn new(data: DataType) -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx,
        _param: InstantiateParam,
    ) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

This example showcases two usages of generics in contract:
- Generic field 
- Generic message parameter

^Sylvia works in both cases, and you can expand your generics to every message type, return type, or `custom_queries` as shown in the case of the `interface`. For the readability of this example, we will
keep just these two generic types.

`InstantiateParam` is passed as a field to the `InstantiateMsg`. This enforces some bounds
to this type, which we pass in the `where` clause. 
Just like that, we created a generic contract.

## Implement generic interface on generic contract

Now that we have the generic contract, let's implement a generic interface on it.
Sorry in advance for the naming of the following file ^^.

`src/generic_generic_impl.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

use crate::generic::Generic;
use crate::generic_contract::GenericContract;
use crate::messages::{MyMsg, MyQuery};

#[contract(module=crate::generic_contract)]
#[messages(crate::generic as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl<DataType, InstantiateParam> Generic<MyMsg, MyMsg, MyQuery, MyMsg>
    for GenericContract<DataType, InstantiateParam>
{
    type Error = StdError;

    #[msg(exec)]
    fn generic_exec(&self, _ctx: ExecCtx<MyQuery>, _param: MyMsg) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }

    #[msg(query)]
    fn generic_query(&self, _ctx: QueryCtx<MyQuery>, _param: MyMsg) -> StdResult<String> {
        Ok(String::default())
    }
}
```

Implementing a generic interface on the generic contract is very similar to implementing it on a regular one.
The only change is to pass the generics into the contract.

Now we will have to change the `GenericContract`. If the interface wasn't `custom` we could just
add the `messages` attribute to it, but because we cannot implement `custom` interface on `non custom`
contract, as explained in the `custom` chapter, we have to change the contract to be `custom`.

```rust
use crate::messages::{MyMsg, MyQuery};
use cosmwasm_std::{CustomMsg, Response, StdResult};
use serde::de::DeserializeOwned;
use std::marker::PhantomData;
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<'static, DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[contract]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + DeserializeOwned + 'static,
    for<'data> DataType: 'data,
{
    pub fn new(data: DataType) -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<MyQuery>,
        _param: InstantiateParam,
    ) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }
}
```

Now that the contract is set as `custom` we can add the `messages` attribute to it.

```rust
use crate::messages::{MyMsg, MyQuery};
use cosmwasm_std::{CustomMsg, Response, StdResult};
use serde::de::DeserializeOwned;
use std::marker::PhantomData;
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<'static, DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[contract]
#[messages(crate::generic<MyMsg, MyMsg, MyQuery, MyMsg> as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + DeserializeOwned + 'static,
    for<'data> DataType: 'data,
{
    pub fn new(data: DataType) -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<MyQuery>,
        _param: InstantiateParam,
    ) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }
}
```

Cool. The contract is almost ready. We are missing just one thing.

## Generate entry points

Without the entry points, our contract is just a library defining some message types.
Let's make it a proper contract.
We have to pass solid types to the `entry_points` macro, as the contract cannot have a generic
types in the entry points.
To achieve this, we pass the solid types to the `entry_points` macro call.

```rust
use crate::messages::{MyMsg, MyQuery};
use cosmwasm_std::{CustomMsg, Response, StdResult};
use cw_storage_plus::Item;
use serde::de::DeserializeOwned;
use std::marker::PhantomData;
use sylvia::types::InstantiateCtx;
use sylvia::{contract, entry_points};

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<'static, DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[entry_points(generics<String, MyMsg>)]
#[contract]
#[messages(crate::generic<MyMsg, MyMsg, MyQuery, MyMsg> as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + DeserializeOwned + 'static,
    for<'data> DataType: 'data,
{
    pub fn new() -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<MyQuery>,
        _param: InstantiateParam,
    ) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }
}
```

Our contract is ready to use. We can test it in the last step of this chapter.

## Test generic contract

^Sylvia enforces the user to specify a solid type while implementing a generic interface on the contract.
Due to this, we test `NonGenericContract` as a regular contract.

In case of the generic contract, we have to pass the types while constructing `CodeId`.

`src/contract.rs`
```rust
use crate::messages::{MyMsg, MyQuery};
use cosmwasm_std::{CustomMsg, Response, StdResult};
use cw_storage_plus::Item;
use serde::de::DeserializeOwned;
use std::marker::PhantomData;
use sylvia::types::InstantiateCtx;
use sylvia::{contract, entry_points};

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<'static, DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[entry_points(generics<String, MyMsg>)]
#[contract]
#[messages(crate::generic<MyMsg, MyMsg, MyQuery, MyMsg> as Generic)]
#[sv::custom(msg=MyMsg, query=MyQuery)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + DeserializeOwned + 'static,
    for<'data> DataType: 'data,
{
    pub fn new() -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx<MyQuery>,
        _param: InstantiateParam,
    ) -> StdResult<Response<MyMsg>> {
        Ok(Response::new())
    }
}

#[cfg(test)]
mod tests {
    use sylvia::multitest::App;

    use crate::messages::{MyMsg, MyQuery};

    use super::sv::multitest_utils::CodeId;

    #[test]
    fn generic_contract() {
        let app = App::<cw_multi_test::BasicApp<MyMsg, MyQuery>>::custom(|_, _, _| {});
        let code_id: CodeId<String, _, _> = CodeId::store_code(&app);

        let owner = "owner";

        let _ = code_id
            .instantiate(MyMsg {})
            .with_label("GenericContract")
            .with_admin(owner)
            .call(owner)
            .unwrap();
    }
}
```

We have to create the `App` by inserting `cw_multi_test::BasicApp` to it as it was explained
in the `custom` chapter.
Because `custom_msg` is defined as one of the generic types we have to pass only one type
to the `CodeId`, which is `DataType`.
It seems strange that `CodeId` is generic over three types while we defined only two,
but `CodeId` is generic over `cw_multi_test::App`. The compiler will always deduce this type from
the `app` passed to it, so don't worry and pass a placeholder there.

Perfect! We have learned how to create generic contracts and interfaces.
Good luck applying this knowledge to your projects!
