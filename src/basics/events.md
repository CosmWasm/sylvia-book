# Events attributes and data

The only way our contract can communicate to the world, for now, is by queries. Smart contracts are
passive - they cannot invoke any action by themselves. They can do it only as a reaction to a call.
But if you tried playing with wasmd, you know that execution on the blockchain can return some
metadata.

There are two things the contract can return to the caller: events and data. Events are something
produced by almost every real-life smart contract. In contrast, data is rarely used, designed for
contract-to-contract communication.

## Returning events

As an example, we would add an event  admin_added emitted by our contract on the execution of
add_member:

```rust,noplayground
use crate::error::ContractError;
use crate::responses::AdminListResp;
use cosmwasm_std::{
    Addr, Deps, DepsMut, Empty, Env, Event, MessageInfo, Order, Response, StdResult,
};
use cw_storage_plus::Map;
use schemars;
use sylvia::contract;

#pub struct AdminContract<'a> {
#    pub(crate) admins: Map<'a, &'a Addr, Empty>,
#}
#
#[contract]
impl AdminContract<'_> {
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
    ...

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

        let resp = Response::new()
            .add_attribute("action", "add_member")
            .add_event(Event::new("admin_added").add_attribute("addr", admin));
        Ok(resp)
    }
}
#
# #[cfg(test)]
#mod tests {
#    use crate::{execute, instantiate, query};
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

An event is built from two things: an event type provided in the
[`new`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Event.html#method.new) function and
attributes. Attributes are added to an event with the
[`add_attributes`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Event.html#method.add_attributes)
or the [`add_attribute`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Event.html#method.add_attribute)
call. Attributes are key-value pairs. Because an event cannot contain any list, to achieve reporting
multiple similar actions taking place, we need to emit multiple small events instead of a collective
one.

Events are emitted by adding them to the response with
[`add_event`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Response.html#method.add_event)
or [`add_events`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Response.html#method.add_events)
call. Additionally, there is a possibility to add attributes directly to the response. It is just
sugar. By default, every execution emits a standard "wasm" event. Adding attributes to the result
adds them to the default event.

We can check if events are properly emitted by contract. It is not always done, as it is much of
boilerplate in test, but events are, generally, more like logs - not necessarily considered the main
contract logic.

Due to the extra event attribute being added by contract, let's add a new method to the proxy:

```rust,noplayground
use cosmwasm_std::{Addr, StdResult};
use cw_multi_test::{App, AppResponse, Executor};

use crate::{
    contract::{AdminContract, ExecMsg, InstantiateMsg, QueryMsg},
    error::ContractError,
    responses::AdminListResp,
};

# #[derive(Clone, Copy, Debug, PartialEq, Eq)]
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
#
# #[derive(Debug)]
#pub struct AdminContractProxy(Addr);
#
impl AdminContractProxy {
    pub fn addr(&self) -> &Addr {
        &self.0
    }
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
}
```

This will return the address of the contract on a blockchain which is, by default, added to the
events attribute and which we will need in our assertion. Now let's update our basic multitest
checking if execution emits events:

```rust,noplayground/If
use cosmwasm_std::{Addr, Event};
use cw_multi_test::App;

use crate::error::ContractError;
use crate::{multitest::proxy::AdminContractCodeId, responses::AdminListResp};

#[test]
fn basic() {
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

    let resp = contract
        .add_member(&mut app, &admin1, admin3.to_string())
        .unwrap();

    let wasm = resp.events.iter().find(|ev| ev.ty == "wasm").unwrap();
    assert_eq!(
        wasm.attributes
            .iter()
            .find(|attr| attr.key == "action")
            .unwrap()
            .value,
        "add_member"
    );

    let admin_added: Vec<_> = resp
        .events
        .iter()
        .filter(|ev| ev.ty == "wasm-admin_added")
        .collect();
    assert_eq!(admin_added[0], &Event::new("wasm-admin_added").add_attribute("_contract_addr", contract.addr()).add_attribute("addr", Addr::unchecked("admin3")));

    let resp = contract.admin_list(&app).unwrap();

    assert_eq!(
        resp,
        AdminListResp {
            admins: vec![admin1.to_string(), admin2.to_string(), admin3.to_string()]
        }
    );
}
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
```

If you have prepared an Ok scenario test after the previous chapter, then you can update it as
above. I have added the `add_member` call in the `basic` test and captured the response in the
`resp` variable.

As you can see, testing events on a simple test made it clunky. First of all, every string is
heavily string-based - a lack of type control makes writing such tests difficult. Also, event types
are prefixed with "wasm-" - it may not be a huge problem, but it doesn't clarify verification. But
the problem is, how layered event structures are, which makes verifying them tricky. Also, the
"wasm" event is particularly tricky, as it contains an implied attribute - `_contract_addr`
containing an address called a contract. My general rule is - do not test emitted events unless
some logic depends on them.

## Data

Besides events, any smart contract execution may produce a `data` object. In contrast to events,
`data` can be structured. It makes it a way better choice to perform any communication logic to
rely on it. On the other hand, it turns out it is very rarely helpful outside of contract-to-contract
communication. Data is always only one single object on the response, which is set using the
[`set_data`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/struct.Response.html#method.set_data)
function. Because of its low usefulness in a single contract environment, we will not spend time on
it right now - an example of it will be covered later when contract-to-contract communication will
be discussed. Until then, it is just helpful to know such an entity exists.
