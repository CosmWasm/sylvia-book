# Testing a query

Last time we created a new query, now it is time to test it out. We will start with the basics -
the unit test. This approach is simple and doesn't require much knowledge besides Rust. Go to the
`src/contract.rs` and add a test on bottom of the file:

```rust,noplayground
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
#    ) -> StdResult<Response> {
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
#}

#[cfg(test)]
mod tests {
    use crate::entry_points::{instantiate, query};
    use cosmwasm_std::from_binary;
    use cosmwasm_std::testing::{mock_dependencies, mock_env, mock_info};

    use super::*;

    #[test]
    fn admin_list_query() {
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

        let msg = QueryMsg::AdminList {};
        let resp = query(deps.as_ref(), env, ContractQueryMsg::AdminContract(msg)).unwrap();
        let resp: AdminListResp = from_binary(&resp).unwrap();
        assert_eq!(
            resp,
            AdminListResp {
                admins: vec!["admin1".to_owned(), "admin2".to_owned()],
            }
        );
    }
}
```

We have a simple flow here:
- we instantiate contract with admins ["admin1", "admin2"],
- we query contract to see if both of them will be returned

Our `instantiate` and `query` require deps and env as parameters. We will mock them with [`mock_dependencies`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/testing/fn.mock_dependencies.html)
and [`mock_env`](https://docs.rs/cosmwasm-std/1.1.0/cosmwasm_std/testing/fn.mock_env.html) from `comswasm-std`.

You may notice the dependencies mock is of a type
[`OwnedDeps`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html) instead
of `Deps`, which we need here - this is why the
[`as_ref`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html#method.as_ref)
function is called on it. If we looked for a `DepsMut` object, we would use
[`as_mut`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/struct.OwnedDeps.html#method.as_mut)
instead.

I extracted the `deps` and `env` variables to their variables
and passed them to calls. The idea is that those variables represent some blockchain persistent state,
and we don't want to create them for every call. We want any changes to the contract state occurring
in `instantiate` to be visible in the `query`. Also, we want to control how the environment differs
on the query and instantiation.

The `info` argument is another story. The message info is unique for each message sent. To create the
`info` mock, we must pass two arguments to the
[`mock_info`](https://docs.rs/cosmwasm-std/1.0.0/cosmwasm_std/testing/fn.mock_info.html) function.

First is the address performing a call. It may look strange to pass `sender` as an address instead of some
mysterious `wasm` followed by hash, but it is a valid address. For testing purposes, such addresses are
typically better, as they are way more verbose in case of failing tests.

The second argument is funds sent with the message. For now, we leave it as an empty slice, as I don't want
to talk about token transfers yet - we will cover it later.

So now it is more a real-case scenario. I see just one problem. Nothing connects the `instantiate` call to the corresponding `query`. It seems that we assume
there is some global contract. But it seems that if we would like to have two contracts instantiated differently
in a single test case, it would become a mess. If only there would be some tool to abstract this for us, wouldn't
it be nice?

