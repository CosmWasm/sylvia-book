# Introducing multitest

Let me introduce the [`multitest`](https://crates.io/crates/cw-multi-test) -
library for creating tests for smart contracts in Rust.

The core idea of `multitest` is abstracting an entity of contract and
simulating the blockchain environment for testing purposes. The purpose of this
is to be able to test communication between smart contracts. It does its job
well, but it is also an excellent tool for testing single-contract scenarios.

## Update dependencies

First, we need to add a [`cw-multi-test`](https://docs.rs/cw-multi-test/latest/cw_multi_test/) to
our `Cargo.toml`. We will also add [`anyhow`](https://docs.rs/anyhow/1.0.66/anyhow/). We will use it
to bail on calls to unimplemented entry points like `reply`, `migrate` and `sudo`.

```toml
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
thiserror = "1.0.37"
cw-storage-plus = "0.16.0"

[dev-dependencies]
anyhow = "1"
cw-multi-test = "0.16"
```

I added a new
[`[dev-dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#development-dependencies)
section with dependencies not used by the final binary
but which may be used by tools around the development process - for example, tests.

## Creating module for tests

Now we will create a new module multitest. Let's first add it to `src/lib.rs`

```rust,noplayground
pub mod contract;
pub mod responses;

#[cfg(test)]
mod multitest;

#use cosmwasm_std::{entry_point, Binary, Deps, DepsMut, Env, MessageInfo, Response};
#
#use crate::contract::{AdminContract, ContractQueryMsg, InstantiateMsg};
#
#const CONTRACT: AdminContract = AdminContract::new();
#
# #[entry_point]
#pub fn instantiate(
#    deps: DepsMut,
#    env: Env,
#    info: MessageInfo,
#    msg: InstantiateMsg,
#) -> StdResult<Response> {
#    msg.dispatch(&CONTRACT, (deps, env, info))
#}
#
# #[entry_point]
#pub fn query(deps: Deps, env: Env, msg: ContractQueryMsg) -> StdResult<Binary> {
#    msg.dispatch(&CONTRACT, (deps, env))
#}
```

As this module is purely for testing purpose we prefix it with
[`#[cfg(test)]`](https://doc.rust-lang.org/book/ch11-03-test-organization.html#the-tests-module-and-cfgtest).

Now create `src/multitest.rs`.

```rust,noplayground
use anyhow::{bail, Result as AnyResult};
use cosmwasm_std::{from_slice, Empty};
use cw_multi_test::Contract;

use crate::contract::{AdminContract, ContractExecMsg, ContractQueryMsg, InstantiateMsg};

mod proxy;
mod tests;

impl Contract<Empty> for AdminContract<'_> {
    fn execute(
        &self,
        deps: cosmwasm_std::DepsMut<Empty>,
        env: cosmwasm_std::Env,
        info: cosmwasm_std::MessageInfo,
        msg: Vec<u8>,
    ) -> AnyResult<cosmwasm_std::Response<Empty>> {
        from_slice::<ContractExecMsg>(&msg)?
            .dispatch(self, (deps, env, info))
            .map_err(Into::into)
    }

    fn instantiate(
        &self,
        deps: cosmwasm_std::DepsMut<Empty>,
        env: cosmwasm_std::Env,
        info: cosmwasm_std::MessageInfo,
        msg: Vec<u8>,
    ) -> AnyResult<cosmwasm_std::Response<Empty>> {
        from_slice::<InstantiateMsg>(&msg)?
            .dispatch(self, (deps, env, info))
            .map_err(Into::into)
    }

    fn query(
        &self,
        deps: cosmwasm_std::Deps<Empty>,
        env: cosmwasm_std::Env,
        msg: Vec<u8>,
    ) -> AnyResult<cosmwasm_std::Binary> {
        from_slice::<ContractQueryMsg>(&msg)?
            .dispatch(self, (deps, env))
            .map_err(Into::into)
    }

    fn sudo(
        &self,
        _deps: cosmwasm_std::DepsMut<Empty>,
        _env: cosmwasm_std::Env,
        _msg: Vec<u8>,
    ) -> AnyResult<cosmwasm_std::Response<Empty>> {
        bail!("sudo not implemented for contract")
    }

    fn reply(
        &self,
        _deps: cosmwasm_std::DepsMut<Empty>,
        _env: cosmwasm_std::Env,
        _msg: cosmwasm_std::Reply,
    ) -> AnyResult<cosmwasm_std::Response<Empty>> {
        bail!("reply not implemented for contract")
    }

    fn migrate(
        &self,
        _deps: cosmwasm_std::DepsMut<Empty>,
        _env: cosmwasm_std::Env,
        _msg: Vec<u8>,
    ) -> AnyResult<cosmwasm_std::Response<Empty>> {
        bail!("reply not implemented for contract")
    }
}
```

So first we added `mod proxy` and `mod tests`. We will add these files in next step so we can
already add them here.

Next we will impl [`Contract`](https://docs.rs/cw-multi-test/0.16.1/cw_multi_test/trait.Contract.html)<Empty>
for `AdminContract`.
This will allow us to use it in cw-multi-test environment. To use it we have to implement six entry
points, but currently our contract supports only two of them `instantiate` and `query`.
For unsuppoprted entry_points we will just call [`bail!`](https://docs.rs/anyhow/latest/anyhow/macro.bail.html).
For supported ones we will handle them simmiliary as we did in `src/lib.rs`.
Difference here is that interface `Contract` force us to pass messages to entry points as binary slices.
We can work with this by using [`from_slice`](https://docs.rs/cosmwasm-std/0.16.0/cosmwasm_std/fn.from_slice.html).
This function will parse binary slice to our messages.
We will then dispatch them like we did in `src/lib.rs` entry points and `map_err` in case any error
will be returned.

## Prepare proxy

Now we will prepare proxy for our contract. Our goal here is to remove repetitiveness and hide
serialization from our tests.

Create `src/multitest/proxy.rs` and paste to it:

```rust,noplayground
use cosmwasm_std::{Addr, StdResult};
use cw_multi_test::{App, Executor};

use crate::{
    contract::{AdminContract, InstantiateMsg, QueryMsg},
    responses::AdminListResp,
};

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct AdminContractCodeId(u64);

impl AdminContractCodeId {
    pub fn store_code(app: &mut App) -> Self {
        let code_id = app.store_code(Box::new(AdminContract::new()));
        Self(code_id)
    }

    #[track_caller]
    pub fn instantiate(
        self,
        app: &mut App,
        sender: &Addr,
        admins: Vec<String>,
        label: &str,
        admin: Option<String>,
    ) -> StdResult<AdminContractProxy> {
        let msg = InstantiateMsg { admins };

        app.instantiate_contract(self.0, sender.clone(), &msg, &[], label, admin)
            .map_err(|err| err.downcast().unwrap())
            .map(AdminContractProxy)
    }
}

#[derive(Debug)]
pub struct AdminContractProxy(Addr);

impl AdminContractProxy {
    #[track_caller]
    pub fn admin_list(&self, app: &App) -> StdResult<AdminListResp> {
        let msg = QueryMsg::AdminList {};

        app.wrap().query_wasm_smart(self.0.clone(), &msg)
    }
}

```

Two new structures here: `AdminContractCodeId` and `AdminContractProxy`.
`AdminContractCodeId` will store `u64` which represents code id of our contract registered on
blockchain generated by [`store_code`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.App.html#method.store_code).
We are able to use [`instantiate_contract`](https://docs.rs/cw-multi-test/latest/cw_multi_test/trait.Executor.html#method.instantiate_contract)
with aquired code id and map address of received contract `addr`.
Then we will define methods in `AdminContractProxy` to call queries and executes on contract.

## First multitest

Now we are ready to write our first multitest. Let's proceed with creating `src/multitest/tests.rs`.

```rust,noplayground
use cosmwasm_std::Addr;
use cw_multi_test::App;

use crate::{multitest::proxy::AdminContractCodeId, responses::AdminListResp};

#[test]
fn basic() {
    let mut app = App::default();

    let owner = Addr::unchecked("addr0001");
    let admins = vec![
        "admin1".to_owned(),
        "admin2".to_owned(),
        "admin3".to_owned(),
    ];

    let code_id = AdminContractCodeId::store_code(&mut app);

    let contract = code_id
        .instantiate(&mut app, &owner, admins.clone(), "Cw20 contract", None)
        .unwrap();

    let resp = contract.admin_list(&app).unwrap();

    assert_eq!(resp, AdminListResp { admins });
}
```

We will first create default [`App`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.App.html#).
This will cache the state of our contract. We will create owner of our contract using
[`Addr::unchecked`](https://docs.rs/cosmwasm-std/0.14.0/cosmwasm_std/struct.Addr.html#method.unchecked).
Call store_code on our `App` to aquire code id and then init the contract aquiring `AdminContractProxy`.
It will be instantiated with three admins which we will then query using `admin_list` method on proxy.
This should return to us `AdminListResp` with all three of them.

The test should pass and we should have our first multitest. We will later expand it when we will
have more functionality to test.
Let's allow our contract to change it's state using `execute` message in next chapter.
