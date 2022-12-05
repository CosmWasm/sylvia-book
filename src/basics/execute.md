# Execution messsages

We created instantiate and query messages. We have state on our contract and are able to test it. Now let's expand our contract by adding to it possibility to update the state. In this chapter we will add `add_member` execute message.

## Custom error

Because we don't want non admins to add new admins to our contract we will have to take some steps to prevent it. In case of call from non admin we want to return an error that will inform the users that they are not authorized to perform this kind of operation on contract.
We will achieve this goal by creating our own custom error type. It will have to be able to be constructed from `StdError`. Another variant of it will `Unathorized`.

First let's update our `Cargo.toml` with new dependency to [`thiserror`](https://docs.rs/thiserror/latest/thiserror/).

```rust,noplayground
[package]
name = "contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[features]
library = []

[dependencies]
cosmwasm-std = { version = "1.1", features = ["staking"] }
cosmwasm-schema = "1.1.6"
serde = { version = "1.0.147", features = ["derive"] }
sylvia = "0.2.1"
schemars = "0.8.11"
cw-storage-plus = "0.16.0"
thiserror = "1.0.37"

[dev-dependencies]
anyhow = "1"
cw-multi-test = "0.16"
```

This error provides us with a derive macro which we will use to implement our `ContractError`.

`src/error.rs`

```rust,noplayground
use cosmwasm_std::{Addr, StdError};
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),
    #[error("{sender} is not a contract admin")]
    Unauthorized { sender: Addr },
}
```

Our custom error will derive

- [`Error`](https://docs.rs/thiserror/latest/thiserror/derive.Error.html)
- `Debug` - for testing purposes
- `PartialEq` - for testing purposes
  Thanks to `#[error(_)]` we will generate Display implementation for our variants. In case of `StdError` we want only to forward the error message. In case of our custom `Unauthorized` variant user will receive information about who sent the message and why it failed.

## Impl execute message

With error created let's implement the message.
It will add new admin if he's not already on the list and if sender is an admin.

```rust,noplayground
use crate::error::ContractError;
use crate::responses::AdminListResp;
use cosmwasm_std::{Addr, Deps, DepsMut, Empty, Env, MessageInfo, Order, Response, StdResult};
use cw_storage_plus::Map;
use schemars;
use sylvia::contract;

#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#}

#[contract]
impl AdminContract<'_> {
#    pub const fn new() -> Self {
#         Self {
#            admins: Map::new("admins"),
#        }
#    }
#
#    #[msg(instantiate)]
#    pub fn instantiate(
#        &self,
#        ctx: (DepsMut, Env, MessageInfo),
#        admins: Vec<String>,
#    ) -> Result<Response, ContractError> {
#        let (deps, _, _) = ctx;
#
#        for admin in admins {
#            let admin = deps.api.addr_validate(&admin)?;
#            self.admins.save(deps.storage, &admin, &Empty {})?;
#        }
#
#        Ok(Response::new())
#    }
#
#    #[msg(query)]
#    pub fn admin_list(&self, ctx: (Deps, Env)) -> StdResult<AdminListResp> {
#        let (deps, _) = ctx;
#
#        let admins: Result<_, _> = self
#            .admins
#            .keys(deps.storage, None, None, Order::Ascending)
#            .map(|addr| addr.map(String::from))
#            .collect();
#
#        Ok(AdminListResp { admins: admins? })
#    }

    #[msg(exec)]
    pub fn add_member(
        &self,
        ctx: (DepsMut, Env, MessageInfo),
        admin: String,
    ) -> Result<Response, ContractError> {
        let (deps, _, info) = ctx;

        if !self.admins.has(deps.storage, &info.sender) {
            return Err(ContractError::Unauthorized {
                sender: info.sender,
            });
        }
        let admin = deps.api.addr_validate(&admin)?;
        self.admins.save(deps.storage, &admin, &Empty {})?;

        Ok(Response::new())
    }
}

##[cfg(test)]
#mod tests {
#    use crate::entry_points::{instantiate, query};
#    use cosmwasm_std::from_binary;
#    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};
#
#    use super::*;
#
#    #[test]
#    fn admin_list_query() {
#        let mut deps = mock_dependencies();
#        let env = mock_env();
#
#        instantiate(
#            deps.as_mut(),
#            env.clone(),
#            mock_info("sender", &[]),
#            InstantiateMsg {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#            },
#        )
#        .unwrap();
#
#        let msg = QueryMsg::AdminList {};
#        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
#        let resp: AdminListResp = from_binary(&resp).unwrap();
#        assert_eq!(
#            resp,
#            AdminListResp {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#            }
#        );
#    }
#}
```

First let's add module `ContractError` to `src/contract.rs`. We will update `instantiate` to return it instead of `StdError`. In case of `query` it is mostly not neccessary as rarely we will check anything in it, but if you will have a reason you can also update it. It is a good approach to define your own error type and return it in all but `query` messages.

To generate message as execute we will prefix it with `#[msg(exec)]`. Return type is the same as in case of `instantiate` which is `Result<Response, ContractError>`.
To check if sender is present in `admins` map we use [`has`](https://docs.rs/cw-storage-plus/0.16.0/cw_storage_plus/struct.Map.html#method.has). We will aquire sender from `MessageInfo`. It will return false in case sender is unathorized and we will return its Addr in `ContractError::Unauthorized`.
Because we can't be sure if address sent by the admin is correct and represent actual `Addr` in blockchain we must first `addr_validate` it. If it's correct we can check if this address is not already in our state.
If it is we will return `Ok(Response)` as nothing bad happened here. If it was not present we can now save it and return `Ok(Response)`.

## Update entry points

Now that our `ExecMsg` is created let's create new entry point for it. We have to do it only once per every type of message thanks to the dispatch method.

```rust,noplayground
pub mod contract;
pub mod error;
pub mod responses;

#[cfg(test)]
mod multitest;

#[cfg(not(feature = "library"))]
mod entry_points {
    use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response};

    use crate::contract::{AdminContract, ContractExecMsg, ContractQueryMsg, InstantiateMsg};
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

    #[entry_point]
    pub fn query(deps: Deps, env: Env, msg: ContractQueryMsg) -> Result<Binary, ContractError> {
        msg.dispatch(&CONTRACT, (deps, env))
    }

    #[entry_point]
    pub fn execute(
        deps: DepsMut,
        env: Env,
        info: MessageInfo,
        msg: ContractExecMsg,
    ) -> Result<Response, ContractError> {
        msg.dispatch(&CONTRACT, (deps, env, info))
    }
}
```

Nothing new here. We have the same `deps`, `env` and `info` variables in signature as in case of `instantiate`.
Our message is `ContractExecMsg` similiar to `ContractQueryMsg` in case of `query`. Body of the function is simple dispatch called on `msg` variable.
We also updated instantiate to return `ContractError`.

## Unit testing

Now let's add simple unit test for execute message.

```rust,noplayground
#use crate::error::ContractError;
#use crate::responses::AdminListResp;
#use cosmwasm_std::{Addr, Deps, DepsMut, Empty, Env, MessageInfo, Order, Response, StdResult};
#use cw_storage_plus::Map;
#use schemars;
#use sylvia::contract;
#
#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#}
#
##[contract]
#impl AdminContract<'_> {
#    pub const fn new() -> Self {
#        Self {
#            admins: Map::new("admins"),
#        }
#    }
#
#    #[msg(instantiate)]
#    pub fn instantiate(
#        &self,
#        ctx: (DepsMut, Env, MessageInfo),
#        admins: Vec<String>,
#    ) -> Result<Response, ContractError> {
#        let (deps, _, _) = ctx;
#
#        for admin in admins {
#            let admin = deps.api.addr_validate(&admin)?;
#            self.admins.save(deps.storage, &admin, &Empty {})?;
#        }
#
#        Ok(Response::new())
#    }
#
#    #[msg(query)]
#    pub fn admin_list(&self, ctx: (Deps, Env)) -> StdResult<AdminListResp> {
#        let (deps, _) = ctx;
#
#        let admins: Result<_, _> = self
#            .admins
#            .keys(deps.storage, None, None, Order::Ascending)
#            .map(|addr| addr.map(String::from))
#            .collect();
#
#        Ok(AdminListResp { admins: admins? })
#    }
#
#    #[msg(exec)]
#    pub fn add_member(
#        &self,
#        ctx: (DepsMut, Env, MessageInfo),
#        admin: String,
#    ) -> Result<Response, ContractError> {
#        let (deps, _, info) = ctx;
#
#        if !self.admins.has(deps.storage, &info.sender) {
#            return Err(ContractError::Unauthorized {
#                sender: info.sender,
#            });
#        }
#        let admin = deps.api.addr_validate(&admin)?;
#
#        self.admins.save(deps.storage, &admin, &Empty {})?;
#
#        Ok(Response::new().add_attribute("action", "add_member"))
#    }
#}
#
##[cfg(test)]
#mod tests {
#    use crate::entry_points::{execute, instantiate, query};
#    use cosmwasm_std::from_binary;
#    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};
#
#    use super::*;
#
#    #[test]
#    fn admin_list_query() {
#        let mut deps = mock_dependencies();
#        let env = mock_env();
#
#        instantiate(
#            deps.as_mut(),
#            env.clone(),
#            mock_info("sender", &[]),
#            InstantiateMsg {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#            },
#        )
#        .unwrap();
#
#        let msg = QueryMsg::AdminList {};
#        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
#        let resp: AdminListResp = from_binary(&resp).unwrap();
#        assert_eq!(
#            resp,
#            AdminListResp {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#            }
#        );
#    }
#
    #[test]
    fn add_member() {
        let mut deps = mock_dependencies();
        let env = mock_env();

        instantiate(
            deps.as_mut(),
            env.clone(),
            mock_info("sender", &[]),
            InstantiateMsg {
                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
            },
        )
        .unwrap();

        let info = mock_info("admin1", &[]);
        let msg = ExecMsg::AddMember {
            admin: "admin3".to_owned(),
        };
        execute(deps.as_mut(), env.clone(), info, ContractExecMsg::AdminContract(msg)).unwrap();

        let msg = QueryMsg::AdminList {};
        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
        let resp: AdminListResp = from_binary(&resp).unwrap();
        assert_eq!(
            resp,
            AdminListResp {
                admins: vec!["admin1".to_owned(), "admin2".to_owned(), "admin3".to_owned()],
            }
        );
    }
}
```

This very similiar to query test. The difference here will be to call `execute` entry point with `ContractExecMsg`. We created another `mock_info` instead of reusing one from instantiate because in real life scenario `MessageInfo` is created for every message.

## Multitest

We have `ExecMsg` created, entry points is estabilished for it and we have simple unit test checking if Ok case is working. Time to test it in `multitest` enviroment.
First we will update our proxy. This time we only need to add call for `add_member`.

```rust,noplayground
use cosmwasm_std::{Addr, StdResult};
use cw_multi_test::{App, AppResponse, Executor};

use crate::{
    contract::{AdminContract, ExecMsg, InstantiateMsg, QueryMsg},
    error::ContractError,
    responses::AdminListResp,
};

##[derive(Clone, Copy, Debug, PartialEq, Eq)]
#pub struct AdminContractCodeId(u64);
#
#impl AdminContractCodeId {
#    pub fn store_code(app: &mut App) -> Self {
#        let code_id = app.store_code(Box::new(AdminContract::new()));
#        Self(code_id)
#    }
#
#    #[track_caller]
#    pub fn instantiate(
#        self,
#        app: &mut App,
#        sender: &Addr,
#        admins: Vec<String>,
#        label: &str,
#        admin: Option<String>,
#    ) -> Result<AdminContractProxy, ContractError> {
#        let msg = InstantiateMsg { admins };
#
#        app.instantiate_contract(self.0, sender.clone(), &msg, &[], label, admin)
#            .map_err(|err| err.downcast().unwrap())
#            .map(AdminContractProxy)
#    }
#}

#[derive(Debug)]
pub struct AdminContractProxy(Addr);

impl AdminContractProxy {
#    #[track_caller]
#    pub fn admin_list(&self, app: &App) -> StdResult<AdminListResp> {
#        let msg = QueryMsg::AdminList {};
#
#        app.wrap().query_wasm_smart(self.0.clone(), &msg)
#    }
#
    #[track_caller]
    pub fn add_member(
        &self,
        app: &mut App,
        sender: &Addr,
        admin: String,
    ) -> Result<AppResponse, ContractError> {
        let msg = ExecMsg::AddMember { admin };

        app.execute_contract(sender.clone(), self.0.clone(), &msg, &[])
            .map_err(|err| err.downcast().unwrap())
    }
}
```

You can see that `App` will return new type [`AppResponse`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.AppResponse.html) from cw_multi_test rather then `Response`.
As for the body of this method it is important to pass `sender` as [`execute_contract`](https://docs.rs/cw-multi-test/latest/cw_multi_test/trait.Executor.html#method.execute_contract) requires it.
We will again pass empty slice as funds as we don't want to deal with it right now.
We call `map_err` here trying to [`downcast`](https://docs.rs/anyhow/1.0.66/anyhow/struct.Error.html#method.downcast) the error to `ContractError`.

Now that proxy is ready let's add new multitest.
Our scenario will be error case. Unauthorized user will try to add itself as an admin to the contract which should fail.

```rust,noplayground
use cosmwasm_std::Addr;
use cw_multi_test::App;

use crate::error::ContractError;
use crate::{multitest::proxy::AdminContractCodeId, responses::AdminListResp};

##[test]
#fn basic() {
#    let mut app = App::default();
#
#    let owner = Addr::unchecked("addr0001");
#    let admin1 = Addr::unchecked("admin1");
#    let admin2 = Addr::unchecked("admin2");
#    let admin3 = Addr::unchecked("admin3");
#
#    let code_id = AdminContractCodeId::store_code(&mut app);
#
#    let contract = code_id
#        .instantiate(
#            &mut app,
#            &owner,
#            vec![admin1.to_string(), admin2.to_string()],
#            "Cw20 contract",
#            None,
#        )
#        .unwrap();
#
#    let resp = contract.admin_list(&app).unwrap();
#
#    assert_eq!(
#        resp,
#        AdminListResp {
#            admins: vec![admin1.to_string(), admin2.to_string()]
#        }
#    );
#
#    contract
#        .add_member(&mut app, &admin1, admin3.to_string())
#        .unwrap();
#
#    let resp = contract.admin_list(&app).unwrap();
#
#    assert_eq!(
#        resp,
#        AdminListResp {
#            admins: vec![admin1.to_string(), admin2.to_string(), admin3.to_string()]
#        }
#    );
#}
#
#[test]
fn unathorized() {
    let mut app = App::default();

    let owner = Addr::unchecked("addr0001");
    let admin1 = Addr::unchecked("admin1");
    let admin2 = Addr::unchecked("admin2");
    let admin3 = Addr::unchecked("admin3");

    let code_id = AdminContractCodeId::store_code(&mut app);

    let contract = code_id
        .instantiate(
            &mut app,
            &owner,
            vec![admin1.to_string(), admin2.to_string()],
            "Cw20 contract",
            None,
        )
        .unwrap();

    let resp = contract.admin_list(&app).unwrap();

    assert_eq!(
        resp,
        AdminListResp {
            admins: vec![admin1.to_string(), admin2.to_string()]
        }
    );

    let err = contract
        .add_member(&mut app, &admin3, admin3.to_string())
        .unwrap_err();

    assert_eq!(err, ContractError::Unauthorized { sender: admin3 });

    let resp = contract.admin_list(&app).unwrap();

    assert_eq!(
        resp,
        AdminListResp {
            admins: vec![admin1.to_string(), admin2.to_string()]
        }
    );
}
```

Once again as in case of unit test this is similiar to the test of basic_query.
After contract instantiation we will call `add_member` with `admin3` as a sender and catch the error using [`unwrap_err`](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_err). Then we will use `assert_eq` to check if it is `ContractError::Unauthorized { sender: "admin3" }`.
In the end we will query the contract for list of admins and `admin3` is not on the list.

Great our contract works as expected but there are some more scenarios we could test. I encourage you to think of other edge cases and try to test them by yourself.
Admins can now be added to the contract but some might want to leave this responsibility. Try to add new message `leave` and don't forget to test the new functionality.
