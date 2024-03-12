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

## Generics in interface

Since `0.10.0` we no longer support generics in interfaces.
Sylvia interfaces can be implemented only a single time per contract as otherwise
the messages would overlap. Idiomatic approach in Rust is to use associated types
to handle such cases.

`src/associated.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::interface;
use sylvia::types::{CustomMsg, ExecCtx, QueryCtx};

#[interface]
pub trait Associated {
    type Error: From<StdError>;
    type ExecParam: CustomMsg;
    type QueryParam: CustomMsg;

    #[msg(exec)]
    fn generic_exec(&self, ctx: ExecCtx, param: Self::ExecParam) -> StdResult<Response>;

    #[msg(query)]
    fn generic_query(&self, ctx: QueryCtx, param: Self::QueryParam) -> StdResult<String>;
}
```

Underhood ^sylvia will parse the associated types and generate generic messages as we cannot
use associated types in enums.

## Implement interface on the contract

Implementing an interface with associated types is the same as in case of implementing a regular interface.
We first need to define the type we will assign to the associated type.

`src/messages.rs`
```rust
use cosmwasm_schema::cw_serde;
use cosmwasm_std::CustomMsg;

#[cw_serde]
pub struct MyMsg;

impl CustomMsg for MyMsg {}
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
    pub const fn new() -> Self {
        Self {}
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

We are set and ready to implement the interface.

`src/associated_impl.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::types::{ExecCtx, QueryCtx};

use crate::associated::Associated;
use crate::contract::NonGenericContract;
use crate::messages::MyMsg;

impl Associated for NonGenericContract {
    type Error = StdError;
    type ExecParam = MyMsg;
    type QueryParam = MyMsg;

    fn generic_exec(&self, _ctx: ExecCtx, _param: Self::ExecParam) -> StdResult<Response> {
        Ok(Response::new())
    }

    fn generic_query(&self, _ctx: QueryCtx, _param: Self::QueryParam) -> StdResult<String> {
        Ok(String::default())
    }
}
```

## Update impl contract

Now that we have implemented the interface on our contract, we can inform
main `sylvia::contract` call about it.

`src/contract.rs`
```rust
use crate::messages::MyMsg;
use cosmwasm_std::{Response, StdResult};
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct NonGenericContract;

#[contract]
#[sv::messages(crate::associated<MyMsg, MyMsg> as Associated)]
impl NonGenericContract {
    pub const fn new() -> Self {
        Self {}
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

As in case of regular interface, we have to add the `messages` attribute to the contract.
However because the interface has associated types, we have to pass the types to the `messages`.
We do that by adding them in the `<>` brackets after the path to the interface module.

In this case we passed concrete types, but it is also possible to pass generic types
defined on the contract.
More on that in the next paragraph.

## Generic contract

We have covered how we can allow the users to define a types of an interface during
an interface implementation.

User might want to use a contract inside it's own contract.
In such cases ^sylvia supports the generics on the contracts.

Let us define a new module in which we will define a generic contract.

`src/generic_contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use std::marker::PhantomData;
use sylvia::contract;
use sylvia::types::{CustomMsg, InstantiateCtx};

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[contract]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + 'static,
    for<'data> DataType: 'data,
{
    pub const fn new() -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[sv::msg(instantiate)]
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

^Sylvia works in both cases, and you can expand your generics to every message type, return type,
or `custom_queries` as shown in the case of the `interface`. For the readability of this example,
we will keep just these two generic types.

`InstantiateParam` is passed as a field to the `InstantiateMsg`. This enforces some bounds
to this type, which we pass in the `where` clause. 
Just like that, we created a generic contract.

## Implement interface on generic contract

Now that we have the generic contract, let's implement an interface from previous paragraphs on it.

`src/associated_impl.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

use crate::associated::Associated;
use crate::generic_contract::GenericContract;
use crate::messages::MyMsg;

impl<DataType, InstantiateParam> Associated for GenericContract<DataType, InstantiateParam> {
    type Error = StdError;
    type ExecParam = MyMsg;
    type QueryParam = MyMsg;

    fn generic_exec(&self, _ctx: ExecCtx, _param: Self::ExecParam) -> StdResult<Response> {
        Ok(Response::new())
    }

    fn generic_query(&self, _ctx: QueryCtx, _param: Self::QueryParam) -> StdResult<String> {
        Ok(String::default())
    }
}
```

Only thing missing is to add the `messages` attribute to the contract implementation.
It is the same as in case of the non generic contract so we will skip it.

Implementing a generic interface on the generic contract is very similar to implementing it on a regular one.
The only change is to pass the generics into the contract.

`src/generic_contract.rs`
```rust
#[contract]
#[messages(crate::associated<MyMsg, MyMsg> as Associated)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + 'static,
    for<'data> DataType: 'data,
{
    ..
}
```

## Forwarding generics

User might want to link the generics from the contract with associated types in implemented interface.
To do so we have to simply assign the generics to appropriate associated types.

Let's one more time define a new contract.

`src/forward_contract.rs`
```rust
use cosmwasm_std::{Response, StdResult};
use std::marker::PhantomData;
use sylvia::contract;
use sylvia::types::{CustomMsg, InstantiateCtx};

pub struct ForwardContract<ExecParam, QueryParam> {
    _phantom: PhantomData<(ExecParam, QueryParam)>,
}

#[contract]
impl<ExecParam, QueryParam> ForwardContract<ExecParam, QueryParam>
where
    ExecParam: CustomMsg + 'static,
    QueryParam: CustomMsg + 'static,
{
    pub const fn new() -> Self {
        Self {
            _phantom: PhantomData,
        }
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

The implementation of the interface should look like this:

`src/associated_impl.rs`
```rust
use cosmwasm_std::{Response, StdError, StdResult};
use sylvia::contract;
use sylvia::types::{CustomMsg, ExecCtx, QueryCtx};

use crate::associated::Associated;
use crate::generic_contract::GenericContract;

impl<ExecParam, QueryParam> Associated for ForwardContract<ExecParam, QueryParam>
where
    ExecParam: CustomMsg,
    QueryParam: CustomMsg,
{
    type Error = StdError;
    type ExecParam = ExecParam;
    type QueryParam = QueryParam;

    fn generic_exec(&self, _ctx: ExecCtx, _param: Self::ExecParam) -> StdResult<Response> {
        Ok(Response::new())
    }

    fn generic_query(&self, _ctx: QueryCtx, _param: Self::QueryParam) -> StdResult<String> {
        Ok(String::default())
    }
}
```

And as always, we have to add the `messages` attribute to the contract implementation.

`src/forward_contract.rs`
```rust
#[contract]
#[sv::messages(crate::associated<ExecParam, QueryParam> as Associated)]
impl<ExecParam, QueryParam> ForwardContract<ExecParam, QueryParam>
where
    ExecParam: CustomMsg + 'static,
    QueryParam: CustomMsg + 'static,
{
    ..
}
```

As you can see ^sylvia is very flexible with how the user might want to use it.

## Generate entry points

Without the entry points, our contract is just a library defining some message types.
Let's make it a proper contract.
We have to pass solid types to the `entry_points` macro, as the contract cannot have a generic
types in the entry points.
To achieve this, we pass the concrete types to the `entry_points` macro call.

```rust
use crate::messages::MyMsg;
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use std::marker::PhantomData;
use sylvia::types::{CustomMsg, InstantiateCtx};
use sylvia::{contract, entry_points};

pub struct GenericContract<DataType, InstantiateParam> {
    _data: Item<DataType>,
    _phantom: PhantomData<InstantiateParam>,
}

#[entry_points(generics<String, MyMsg>)]
#[contract]
#[sv::messages(crate::associated<MyMsg, MyMsg> as Associated)]
impl<DataType, InstantiateParam> GenericContract<DataType, InstantiateParam>
where
    InstantiateParam: CustomMsg + 'static,
    for<'data> DataType: 'data,
{
    pub const fn new() -> Self {
        Self {
            _data: Item::new("data"),
            _phantom: PhantomData,
        }
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(
        &self,
        _ctx: InstantiateCtx,
        _param: InstantiateParam,
    ) -> StdResult<Response> {
        Ok(Response::new())
    }
}
```

Notice the `generics` attribute in the `entry_points` macro.
It's goal is to provide us a way to declare the types we want to use in the entry points.
Our contract is ready to use. We can test it in the last step of this chapter.

## Test generic contract

Similar to defining a contract, we have to create a `App` with the `cw_multi

`src/contract.rs`
```rust
...

#[cfg(test)]
mod tests {
    use sylvia::cw_multi_test::IntoAddr;
    use sylvia::multitest::App;

    use crate::messages::MyMsg;

    use super::sv::mt::CodeId;

    #[test]
    fn instantiate_contract() {
        let app = App::default();
        let code_id: CodeId<String, MyMsg, _> = CodeId::store_code(&app);

        let owner = "owner".into_addr();

        let _ = code_id
            .instantiate(MyMsg {})
            .with_label("GenericContract")
            .with_admin(owner.as_str())
            .call(&owner)
            .unwrap();
    }
}
```

While creating the `CodeId` we have to pass the types we want to use in the contract.
It seems strange that `CodeId` is generic over three types while we defined only two,
but `CodeId` is also generic over `MtApp` (`cw-multi-test::App`). The compiler will always deduce this type from
the `app` passed to it, so don't worry and pass a placeholder there.

Perfect! We have learned how to create generic contracts and interfaces.
Good luck applying this knowledge to your projects!
