# Reusability

We have covered almost everything needed to write CosmWasm smart contracts with `sylvia`.
In this last chapter of the `basics` section, I will tell you about the ability to define
[`interfaces`](https://docs.rs/sylvia/latest/sylvia/attr.interface.html) in `sylvia`.

## Problem

Let's say that after creating this contract we start working on another. While planning its
implementation, we notice that its functionality is just a superset of our `AdminContract`.
We could copy all the code to our new contract, but it's creating unnecessary redundancy and would
force us to maintain multiple implementations of the same functionality.

## Solution

`Sylvia` has a feature to reuse already implemented messages and apply them in new contracts.
Clone and open [`sylvia`](https://github.com/CosmWasm/sylvia) repository. Go to
`contracts/cw1-subkeys/src/contract.rs`. You can notice that the `impl` block for
the `Cw1SubkeysContract` is preceded with `#[messages(...)]` attribute.

```rust,noplayground
#[contract]
#[messages(cw1 as Cw1)]
#[messages(whitelist as Whitelist)]
impl Cw1SubkeysContract<'_> {
    ...
}
```

`contract` macro considers both interfaces marked as `messages`, which in our case
are `cw1` and `whitelist`. It then generates `ContractQueryMsg` and `ContractExecMsg` as such:

```rust,noplayground
#[allow(clippy::derive_partial_eq_without_eq)]
#[serde(rename_all = "snake_case", untagged)]
pub enum ContractQueryMsg {
    Cw1(cw1::Cw1QueryMsg),
    Whitelist(whitelist::WhitelistQueryMsg),
    Cw1SubkeysContract(QueryMsg),
}

impl ContractQueryMsg {
    pub fn dispatch(
        self,
        contract: &Cw1SubkeysContract,
        ctx: (cosmwasm_std::Deps, cosmwasm_std::Env),
    ) -> std::result::Result<sylvia::cw_std::Binary, ContractError> {
        const _: () = {
            let msgs: [&[&str]; 3usize] = [
                &cw1::Cw1QueryMsg::messages(),
                &whitelist::WhitelistQueryMsg::messages(),
                &QueryMsg::messages(),
            ];
            sylvia::utils::assert_no_intersection(msgs);
        };
        match self {
            ContractQueryMsg::Cw1(msg) => msg.dispatch(contract, ctx),
            ContractQueryMsg::Whitelist(msg) => msg.dispatch(contract, ctx),
            ContractQueryMsg::Cw1SubkeysContract(msg) => msg.dispatch(contract, ctx),
        }
    }
}
```

We can finally see why we need these `ContractQueryMsg` and `ContractExecMsg` next to our
regular message enums. `Sylvia` generated three tuple variants:

- `Cw1` - which contains query msg defined in `whitelist`;

- `Whitelist`- which contains query msg defined in `cw1`;

- `Cw1SubkeysContract` - which contains query msg defined in our contract.

We use this wrapper to match with the proper variant and then call `dispatch` on this message.
`Sylvia` also ensure that no message overlaps between interfaces and contract so that
contracts API won't break.

## Declaring interface

How are the `interface` messages implemented? `Cw1SubkeysContract` is an excellent example because
it presents two situations:

- `Cw1` - declares a set of functionality that should be supported in implementing this interface
  contract and forces the user to define behavior for them;
- `Whitelist` - same as above, but being primarily implemented for the `Cw1Whitelist` contract, it
  has already implementation defined.

For the latter one, we can either implement it ourselves or reuse it as it was done
in `contract/cw1-subkeys/src/whitelist.rs`. As you can see, we only call a method on `whitelist`
forwarding the arguments passed to the contract. To see the implementation,
you can go to `contract/cw1-whitelist/src/whitelist.rs`. The interface has to be defined as a
`trait` with a call to macro
[`interface`](https://docs.rs/sylvia/latest/sylvia/attr.interface.html).
It currently supports only `execute` and `query` messages. In this case, it is right away
implemented on the `Cw1Whitelist` contract, and this implementation is being reused in
`contract/cw1-subkeys/src/whitelist.rs`.

You might also want to separate the functionalities of your contract in some sets. It is the case
of `Cw1`. It is created as a separate crate and reused in both `Cw1WhitelistContract` and
`Cw1SubkeysContract`. You can check the implementation in `contracts/cw1-subkeys/src/cw1.rs`.
For interface declaration itself, take a look at `contracts/cw1/src/lib.rs`.

## Practice

We now have enough background to create an `interface` ourselves. Let's say we started
working on some other contract and found out that the `donate` functionality would fit in it very
well. Open our `AdminContract` project. We will first create `src/donation.rs`:

```rust,noplayground
use cosmwasm_std::{DepsMut, Env, MessageInfo, Response, StdError};
use sylvia::interface;

#[interface]
pub trait Donation {
    type Error: From<StdError>;

    #[msg(exec)]
    fn donate(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, Self::Error>;
}
```

We add the `interface` attribute macro to the newly created `trait Donation`. If we stopped here, we
will get an error. `interface` forces us to alias Error for the trait. We will declare it as
`From<StdError>` as it is the error from cosmwasm_std, and we will always want our errors to
implement `From` on it. We then declare `donate` returning `Result` with
`Self::Error`, allowing users to define their error types.

Now that the declaration is ready, let's move the `donate` implementation to this file:

```rust,noplayground
use cosmwasm_std::{coins, BankMsg, DepsMut, Env, MessageInfo, Order, Response, StdError};
use sylvia::interface;

use crate::{contract::AdminContract, error::ContractError};

#[interface]
pub trait Donation {
    type Error: From<StdError>;

    #[msg(exec)]
    fn donate(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, Self::Error>;
}

impl Donation for AdminContract<'_> {
    type Error = ContractError;

    fn donate(&self, ctx: (DepsMut, Env, MessageInfo)) -> Result<Response, ContractError> {
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
```

We alias `Error` as `ContractError` and move the whole method from `src/contract.rs`.
Now we only need to add the `Donation` interface to our contract:

```rust,noplayground
use cosmwasm_std::{
    Addr, Deps, DepsMut, Empty, Env, Event, MessageInfo, Order, Response, StdResult,
};
use cw_storage_plus::{Item, Map};
use schemars;
use sylvia::contract;

use crate::{donation, error::ContractError, responses::AdminListResp};

#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#    pub(crate) donation_denom: Item<'static, String>,
#}
#
#[contract]
#[messages(donation as Donation)]
impl AdminContract<'_> {
    ...
#    pub const fn new() -> Self {
#        AdminContract {
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
#        self.admins.save(deps.storage, &admin, &Empty {})?;
#
#        let resp = Response::new()
#            .add_attribute("action", "add_member")
#            .add_event(Event::new("admin_added").add_attribute("addr", admin));
#        Ok(resp)
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
#                donation_denom: "atom".to_owned(),
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
#                donation_denom: "atom".to_owned(),
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
}
```

Now you should have a compilation error while running tests. `src/multitest/proxy.rs` used
`ExecMsg::Donate {}`, but it is not a part of `ExecMsg` defined in `src/contract.rs`.
You can fix it f.e. by changing it to `let msg = crate::donation::ExecMsg::Donate {};` or you
can alias it in the `use` statement and use it as `DonateExecMsg::Donate {}`.

Now all tests should pass. You can run `cargo schema` to check what changed in your contract API.
