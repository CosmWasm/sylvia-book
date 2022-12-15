# Dealing with funds

When you hear smart contracts, you think of blockchain. When you hear blockchain, you often think of
cryptocurrencies. It is not the same, but crypto assets, or as we often call them: tokens, are very
closely connected to the blockchain. CosmWasm has a notion of a native token. Native tokens are
assets managed by the blockchain core instead of smart contracts. Often such assets have some
special meaning, like being used for paying gas fees or staking for consensus algorithm, but can be
just arbitrary assets.

Native tokens are assigned to their owners but can be transferred by their nature. Everything had an
address in the blockchain is eligible to have its native tokens. As a consequence - tokens can be
assigned to smart contracts! Every message sent to the smart contract can have some funds sent
with it. In this chapter, we will take advantage of that and create a way to reward the hard work
performed by admins. We will create a new message - Donate, which anyone can use to donate some
funds to admins, divided equally.

## Preparing messages

Traditionally we need to prepare our messages. We need to create a new ExecuteMsg variant, but
we will also modify the Instantiate message a bit - we need to have some way of defining the
name of a native token we would use for donations. It would be possible to allow users to send any
tokens they want, but we want to simplify things for now.

```rust,noplayground
use crate::error::ContractError;
use crate::responses::AdminListResp;
use cosmwasm_std::{
    Addr, Deps, DepsMut, Empty, Env, Event, MessageInfo, Order, Response, StdResult,
};
use cw_storage_plus::{Map, Item};
use schemars;
use sylvia::contract;

pub struct AdminContract<'a> {
    pub(crate) admins: Map<'a, &'a Addr, Empty>,
    pub(crate) donation_denom: Item<'a, String>,
}

#[contract]
impl AdminContract<'_> {
    pub const fn new() -> Self {
        Self {
            admins: Map::new("admins"),
            donation_denom: Item::new("donation_denom"),
        }
    }

    #[msg(instantiate)]
    pub fn instantiate(
        &self,
        ctx: (DepsMut, Env, MessageInfo),
        admins: Vec<String>,
        donation_denom: String,
    ) -> Result<Response, ContractError> {
        let (deps, _, _) = ctx;

        for admin in admins {
            let admin = deps.api.addr_validate(&admin)?;
            self.admins.save(deps.storage, &admin, &Empty {})?;
        }
        self.donation_denom.save(deps.storage, &donation_denom)?;
        Ok(Response::new())
    }
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
#        let resp = Response::new().add_attribute("action", "add_member");
#        self.admins.save(deps.storage, &admin, &Empty {})?;
#        let resp = resp.add_event(Event::new("admin_added").add_attribute("addr", admin));
#        Ok(resp)
#    }
#
#    #[msg(exec)]
#    pub fn leave(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, ContractError> {
#        let (deps, _, info) = ctx;
#
#        self.admins.remove(deps.storage, &info.sender);
#
#        Ok(Response::new().add_attribute("action", "leave"))
#    }
#}
#
# #[cfg(test)]
#mod tests {
#    use crate::entry_points::{execute, instantiate, query};
#    use cosmwasm_std::from_binary;
#    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};
#
#    use super::*;
#
#    const ATOM: &str = "atom";
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
#                donation_denom: ATOM.to_owned(),
#            },
#
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
#    #[test]
#    fn add_member() {
#        let mut deps = mock_dependencies();
#        let env = mock_env();
#
#        instantiate(
#            deps.as_mut(),
#            env.clone(),
#            mock_info("sender", &[]),
#            InstantiateMsg {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#                donation_denom: ATOM.to_owned(),
#            },
#        )
#        .unwrap();
#
#        let info = mock_info("admin1", &[]);
#        let msg = ExecMsg::AddMember {
#            admin: "admin3".to_owned(),
#        };
#        execute(
#            deps.as_mut(),
#            env.clone(),
#            info,
#            ContractExecMsg::AdminContract(msg),
#        )
#        .unwrap();
#
#        let msg = QueryMsg::AdminList {};
#        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
#        let resp: AdminListResp = from_binary(&resp).unwrap();
#        assert_eq!(
#            resp,
#            AdminListResp {
#                admins: vec![
#                    "admin1".to_owned(),
#                    "admin2".to_owned(),
#                    "admin3".to_owned()
#                ],
#            }
#        )
#    }
#}
```

We have added a new state `donation_denom`, which is of type
[`Item`](https://docs.rs/cw-storage-plus/latest/cw_storage_plus/struct.Item.html). A user has to
pass a new value to instantiate the contract. I will let you fix tests, which should at this point
fail due to missing parameter.

Let's update our `Cargo.toml` with a new dependency to
[`cw-utils`](https://docs.rs/cw-utils/latest/cw_utils/).

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
thiserror = "1.0.37"
cw-storage-plus = "0.16.0"
cw-utils = "0.16"

[dev-dependencies]
anyhow = "1"
cw-multi-test = "0.16"
```

Now let's implement the new `donate` message.

```rust,noplayground
use crate::error::ContractError;
use crate::responses::AdminListResp;
use cosmwasm_std::{
    Addr, Deps, DepsMut, Empty, Env, Event, MessageInfo, Order, Response, StdResult, StdError, BankMsg, coins,
};
use cw_storage_plus::{Map, Item};
use schemars;
use sylvia::contract;

pub struct AdminContract<'a> {
    pub(crate) admins: Map<'a, &'a Addr, Empty>,
    pub(crate) donation_denom: Item<'a, String>,
}

#[contract]
impl AdminContract<'_> {
#    pub const fn new() -> Self {
#        Self {
#            admins: Map::new("admins"),
#            donation_denom: Item::new("donation_denom"),
#        }
#    }
#
#    #[msg(instantiate)]
#    pub fn instantiate(
#        &self,
#        ctx: (DepsMut, Env, MessageInfo),
#        admins: Vec<String>,
#        donation_denom: String,
#    ) -> Result<Response, ContractError> {
#        let (deps, _, _) = ctx;
#
#        for admin in admins {
#            let admin = deps.api.addr_validate(&admin)?;
#            self.admins.save(deps.storage, &admin, &Empty {})?;
#        }
#        self.donation_denom.save(deps.storage, &donation_denom)?;
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
#        let resp = Response::new().add_attribute("action", "add_member");
#        self.admins.save(deps.storage, &admin, &Empty {})?;
#        let resp = resp.add_event(Event::new("admin_added").add_attribute("addr", admin));
#        Ok(resp)
#    }
#
#    #[msg(exec)]
#    pub fn leave(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, ContractError> {
#        let (deps, _, info) = ctx;
#
#        self.admins.remove(deps.storage, &info.sender);
#
#        Ok(Response::new().add_attribute("action", "leave"))
#    }
#
    ...
    #[msg(exec)]
    pub fn donate(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, ContractError> {
        let (deps, _, info) = ctx;

        let denom = self.donation_denom.load(deps.storage)?;
        let admins_len = self
            .admins
            .keys(deps.storage, None, None, Order::Ascending)
            .filter_map(|admin| admin.ok())
            .count();

        let donation = cw_utils::must_pay(&info, &denom)
            .map_err(|err| StdError::generic_err(err.to_string()))?
            .u128();

        let donation_per_admin = donation / (admins_len as u128);

        let admins = self
            .admins
            .keys(deps.storage, None, None, Order::Ascending)
            .filter_map(|admin| admin.ok());

        let messages = admins.into_iter().map(|admin| BankMsg::Send {
            to_address: admin.to_string(),
            amount: coins(donation_per_admin, &denom),
        });

        let resp = Response::new()
            .add_messages(messages)
            .add_attribute("action", "donate")
            .add_attribute("amount", donation.to_string())
            .add_attribute("per_admin", donation_per_admin.to_string());

        Ok(resp)
    }
}
#
# #[cfg(test)]
#mod tests {
#    use crate::entry_points::{execute, instantiate, query};
#    use cosmwasm_std::from_binary;
#    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};
#
#    use super::*;
#
#    const ATOM: &str = "atom";
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
#                donation_denom: ATOM.to_owned(),
#            },
#
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
#    #[test]
#    fn add_member() {
#        let mut deps = mock_dependencies();
#        let env = mock_env();
#
#        instantiate(
#            deps.as_mut(),
#            env.clone(),
#            mock_info("sender", &[]),
#            InstantiateMsg {
#                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
#                donation_denom: ATOM.to_owned(),
#            },
#        )
#        .unwrap();
#
#        let info = mock_info("admin1", &[]);
#        let msg = ExecMsg::AddMember {
#            admin: "admin3".to_owned(),
#        };
#        execute(
#            deps.as_mut(),
#            env.clone(),
#            info,
#            ContractExecMsg::AdminContract(msg),
#        )
#        .unwrap();
#
#        let msg = QueryMsg::AdminList {};
#        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
#        let resp: AdminListResp = from_binary(&resp).unwrap();
#        assert_eq!(
#            resp,
#            AdminListResp {
#                admins: vec![
#                    "admin1".to_owned(),
#                    "admin2".to_owned(),
#                    "admin3".to_owned()
#                ],
#            }
#        );
#    }
#}

```

Sending the funds to another contract is performed by adding bank messages to the response.
The blockchain would expect any message which is returned in contract response as a part of an
execution. This design is related to an actor model implemented by CosmWasm. The whole actor model
will be described in detail later. For now, you can assume this is a way to handle token transfers.
Before sending tokens to admins, we have to calculate the amount of dotation per admin. It is done
by searching funds for an entry describing our donation token and dividing the number of tokens
sent by the number of admins. Note that because the integral division is always rounding down.

As a consequence, it is possible that not all tokens sent as a donation would end up with no admins
accounts. Any leftover would be left on our contract account forever. There are plenty of ways of
dealing with this issue - figuring out one of them would be a great exercise.

The last missing part is updating the ContractError - the must_pay call returns a
[`PaymentError`](https://docs.rs/cw-utils/latest/cw_utils/enum.PaymentError.html)
which we can't convert to our error type yet:

```rust,noplayground
use cosmwasm_std::{Addr, StdError};
use cw_utils::PaymentError;
use thiserror::Error;

#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),
    #[error("{sender} is not a contract admin")]
    Unauthorized { sender: Addr },
    #[error("Payment error: {0}")]
    Payment(#[from] PaymentError)
}
```

As you can see, to handle incoming funds, I used the utility function - I encourage you to take
a look at its [`implementation`](https://docs.rs/cw-utils/0.13.4/src/cw_utils/payment.rs.html#32-39)
\- this would give you a good understanding of how incoming funds are structured in `MessageInfo`.

Now it's time to check if the funds are distributed correctly. The way for that is to write a test.
First let's update `src/multitest/proxy.rs`

```rust,noplayground
use cosmwasm_std::{Addr, Coin, StdResult};
use cw_multi_test::{App, AppResponse, Executor};

use crate::{
    contract::{AdminContract, ExecMsg, InstantiateMsg, QueryMsg},
    error::ContractError,
    responses::AdminListResp,
};
#
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
#        donation_denom: String,
#        label: &str,
#        admin: Option<String>,
#    ) -> Result<AdminContractProxy, ContractError> {
#        let msg = InstantiateMsg {
#            admins,
#            donation_denom,
#        };
#
#        app.instantiate_contract(self.0, sender.clone(), &msg, &[], label, admin)
#            .map_err(|err| err.downcast().unwrap())
#            .map(AdminContractProxy)
#    }
#}
#
##[derive(Debug)]
#pub struct AdminContractProxy(Addr);
#
impl AdminContractProxy {
#    pub fn addr(&self) -> &Addr {
#        &self.0
#    }
#    #[track_caller]
#    pub fn admin_list(&self, app: &App) -> StdResult<AdminListResp> {
#        let msg = QueryMsg::AdminList {};
#
#        app.wrap().query_wasm_smart(self.0.clone(), &msg)
#    }
#
#    #[track_caller]
#    pub fn add_member(
#        &self,
#        app: &mut App,
#        sender: &Addr,
#        admin: String,
#    ) -> Result<AppResponse, ContractError> {
#        let msg = ExecMsg::AddMember { admin };
#
#        app.execute_contract(sender.clone(), self.0.clone(), &msg, &[])
#            .map_err(|err| err.downcast().unwrap())
#    }
#
#    #[track_caller]
#    pub fn leave(&self, app: &mut App, sender: &Addr) -> Result<AppResponse, ContractError> {
#        let msg = ExecMsg::Leave {};
#
#        app.execute_contract(sender.clone(), self.0.clone(), &msg, &[])
#            .map_err(|err| err.downcast().unwrap())
#    }
#
    #[track_caller]
    pub fn donate(
        &self,
        app: &mut App,
        sender: &Addr,
        funds: &[Coin],
    ) -> Result<AppResponse, ContractError> {
        let msg = ExecMsg::Donate {};

        app.execute_contract(sender.clone(), self.0.clone(), &msg, &funds)
            .map_err(|err| err.downcast().unwrap())
    }
}
```

Now let' add donate test in `src/multitest/tests.rs`

```rust,noplayground
use cosmwasm_std::{coins, Addr, Event};
use cw_multi_test::App;

use crate::error::ContractError;
use crate::{multitest::proxy::AdminContractCodeId, responses::AdminListResp};

const ATOM: &str = "atom";

# #[test]
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
#            ATOM.to_string(),
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
#    let resp = contract
#        .add_member(&mut app, &admin1, admin3.to_string())
#        .unwrap();
#
#    let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
#    assert_eq!(
#        wasm.attributes
#            .iter()
#            .find(|attr| attr.key == "action")
#            .unwrap()
#            .value,
#        "add_member"
#    );
#
#    let admin_added: Vec<_> = resp
#        .events
#        .iter()
#        .filter(|ev| ev.ty == "wasm-admin_added")
#        .collect();
#    assert_eq!(
#        admin_added[0],
#        &Event::new("wasm-admin_added")
#            .add_attribute("_contract_addr", contract.addr())
#            .add_attribute("addr", Addr::unchecked("admin3"))
#    );
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
# #[test]
#fn unathorized() {
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
#            ATOM.to_string(),
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
#    let err = contract
#        .add_member(&mut app, &admin3, admin3.to_string())
#        .unwrap_err();
#
#    assert_eq!(err, ContractError::Unauthorized { sender: admin3 });
#
#    let resp = contract.admin_list(&app).unwrap();
#
#    assert_eq!(
#        resp,
#        AdminListResp {
#            admins: vec![admin1.to_string(), admin2.to_string()]
#        }
#    );
#}
#
# #[test]
#fn leave() {
#    let mut app = App::default();
#
#    let owner = Addr::unchecked("addr0001");
#    let admin1 = Addr::unchecked("admin1");
#    let admin2 = Addr::unchecked("admin2");
#
#    let code_id = AdminContractCodeId::store_code(&mut app);
#
#    let contract = code_id
#        .instantiate(
#            &mut app,
#            &owner,
#            vec![admin1.to_string(), admin2.to_string()],
#            ATOM.to_string(),
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
#    contract.leave(&mut app, &admin1).unwrap();
#
#    let resp = contract.admin_list(&app).unwrap();
#
#    assert_eq!(
#        resp,
#        AdminListResp {
#            admins: vec![admin2.to_string()]
#        }
#    );
#}
#
#[test]
fn donate() {
    let owner = Addr::unchecked("addr0001");
    let admin1 = Addr::unchecked("admin1");
    let admin2 = Addr::unchecked("admin2");

    let mut app = App::new(|router, _, storage| {
        router
            .bank
            .init_balance(storage, &owner, coins(5, ATOM))
            .unwrap()
    });

    let code_id = AdminContractCodeId::store_code(&mut app);

    let contract = code_id
        .instantiate(
            &mut app,
            &owner,
            vec![admin1.to_string(), admin2.to_string()],
            ATOM.to_string(),
            "Cw20 contract",
            None,
        )
        .unwrap();

    contract.donate(&mut app, &owner, &coins(5, ATOM)).unwrap();

    assert_eq!(
        app.wrap().query_balance(owner, ATOM).unwrap().amount.u128(),
        0
    );

    assert_eq!(
        app.wrap()
            .query_balance(contract.addr(), ATOM)
            .unwrap()
            .amount
            .u128(),
        1
    );

    assert_eq!(
        app.wrap()
            .query_balance(admin1, ATOM)
            .unwrap()
            .amount
            .u128(),
        2
    );

    assert_eq!(
        app.wrap()
            .query_balance(admin2, ATOM)
            .unwrap()
            .amount
            .u128(),
        2
    );
}
```

Fairly simple. I don't particularly appreciate that every balance check is eight lines of code,
but it can be improved by enclosing this assertion into a separate function, probably with the
[`#[track_caller]`](https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-track_caller-attribute) attribute.

The critical thing to talk about is how `app` creation changed. Because we need some initial tokens
on an `owner` account, instead of using the default constructor, we have to provide it with an
initializer function. Unfortunately,
[`new`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.App.html#method.new) documentation
is not easy to follow - even if a function is not very complicated. What it takes as an argument is
a closure with three arguments - the
[`Router`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.Router.html)
with all modules supported by multi-test, the API object, and the state. This function is called
once during contract instantiation. The `router` object contains some generic fields
\- we are interested in the bank in particular. It has a type of
[`BankKeeper`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.BankKeeper.html),
where the
[`init_balance`](https://docs.rs/cw-multi-test/latest/cw_multi_test/struct.BankKeeper.html#method.init_balance)
function sits.

## Plot Twist!

As we covered most of the important basics about building Rust smart contracts, I have a serious
exercise for you.

The contract we built has an exploitable bug. All donations are distributed equally across admins.
However, every admin is eligible to add another admin. And nothing is preventing the admin from
adding himself to the list and receiving twice as many rewards as others!

Try to write a test that detects such a bug, then fix it and ensure the bug never more occurs.

Even if the admin cannot add the same address to the list, he can always create new accounts and
add them, but this is something unpreventable on the contract level, so do not prevent that.
Handling this kind of case is done by properly designing whole applications, which is out of this
chapter's scope.
