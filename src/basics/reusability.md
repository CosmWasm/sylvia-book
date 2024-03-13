# Reusability

We have covered almost all the basics of writing smart contracts with ^sylvia.
In this last chapter of the `basics` section, I will tell you about the ability to define
[`interface`](https://docs.rs/sylvia/latest/sylvia/attr.interface.html) in ^sylvia.

## Problem

Let's say that after creating this contract we start working on another one. While planning its
implementation, we notice that its functionality is just a superset of our `AdminContract`.
We could copy all the code to our new contract, but it's creating unnecessary redundancy and would
force us to maintain multiple implementations of the same functionality. It would also mean that 
a bunch of functionality would be crammed together. A better solution would be to divide the code
into semantically compatible parts.

## Solution

^Sylvia has a feature to reuse already defined messages and apply them in new contracts.
Clone and open [^sylvia](https://github.com/CosmWasm/sylvia) repository. Go to
`contracts/cw1-subkeys/src/contract.rs`. You can notice that the `impl` block for
the `Cw1SubkeysContract` is preceded by `#[sv::messages(...)]` attribute.

```rust,noplayground
#[contract]
#[sv::messages(cw1 as Cw1)]
#[sv::messages(whitelist as Whitelist)]
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
regular message enums. ^Sylvia generated three tuple variants:

- `Cw1` - which contains query msg defined in `cw1`;

- `Whitelist`- which contains query msg defined in `whitelist`;

- `Cw1SubkeysContract` - which contains query msg defined in our contract.

We use this wrapper to match with the proper variant and then call `dispatch` on this message.
^Sylvia also ensure that no message overlaps between interfaces and contract so that
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

You should also separate the functionalities of your contract in some sets. It is the case
of `Cw1`. It is created as a separate crate and reused in both `Cw1WhitelistContract` and
`Cw1SubkeysContract`. You can check the implementation in `contracts/cw1-subkeys/src/cw1.rs`.
For interface declaration itself, take a look at `contracts/cw1/src/lib.rs`.

## Practice

We now have enough background to create an `interface` ourselves. Let's say we started
working on some other contract and found out that we would like to restrict access to modify 
the state of the contract. We will create a new `Whitelist` interface which only responsibility will be 
managing a list of admins.
Usually I would suggest switching from a single crate to a workspace repository, but to simplify this
example, I will keep working on a single crate repository.

We want to be able to access the list of `admins` via query. Let's create a new response type 
in `src/responses.rs`:

```rust,noplayground
use cosmwasm_schema::cw_serde;
use cosmwasm_std::Addr;

#[cw_serde]
pub struct CountResponse {
    pub count: u32,
}

#[cw_serde]
pub struct AdminsResponse {
    pub admins: Vec<Addr>,
}
```

We are going to keep the admins as [`Addr`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/struct.Addr.html)
which is a representation of a real address on the blockchain.

Now we create a new module, `src/whitelist.rs` (remember to add it to `src/lib.rs` as public).

```rust,noplayground
use cosmwasm_std::{Response, StdError};
use sylvia::interface;
use sylvia::types::{ExecCtx, QueryCtx};

use crate::responses::AdminsResponse;

#[interface]
pub trait Whitelist {
    type Error: From<StdError>;

    #[sv::msg(exec)]
    fn add_admin(&self, ctx: ExecCtx, address: String) -> Result<Response, Self::Error>;

    #[sv::msg(exec)]
    fn remove_admin(&self, ctx: ExecCtx, address: String) -> Result<Response, Self::Error>;

    #[sv::msg(query)]
    fn admins(&self, ctx: QueryCtx) -> Result<AdminsResponse, Self::Error>;
}
```

We annotate interfaces with [`interface`](https://docs.rs/sylvia/0.7.0/sylvia/attr.interface.html)
attribute macro. It expects us to declare the associated type `Error`. This will help us later as 
otherwise we would have to either expect `StdError` or our custom error in the return type,
but we don't know what contracts will use this interface.

Our trait defines three methods. Let's implement them on our contract.
^Sylvia still grows and there can be some type duplications in the future. I recommend keeping all
three: contract implementation, interface definition and interface implementation on contract in 
separate modules.

Let's then create a new file. It will be called `src/whitelist_impl.rs`.

```rust,noplayground
use cosmwasm_std::{Addr, Empty, Response};
use sylvia::contract;
use sylvia::types::{ExecCtx, QueryCtx};

use crate::contract::CounterContract;
use crate::error::ContractError;
use crate::responses::AdminsResponse;
use crate::whitelist::Whitelist;

#[contract(module=crate::contract)]
#[sv::messages(crate::whitelist as Whitelist)]
impl Whitelist for CounterContract<'_> {
    type Error = ContractError;

    #[sv::msg(exec)]
    fn add_admin(&self, ctx: ExecCtx, admin: String) -> Result<Response, Self::Error> {
        let deps = ctx.deps;
        let admin = deps.api.addr_validate(&admin)?;
        self.admins.save(deps.storage, &admin, &Empty {})?;

        Ok(Response::default())
    }

    #[sv::msg(exec)]
    fn remove_admin(&self, ctx: ExecCtx, admin: String) -> Result<Response, Self::Error> {
        let deps = ctx.deps;
        let admin = deps.api.addr_validate(&admin)?;
        self.admins.remove(deps.storage, &admin);

        Ok(Response::default())
    }

    #[sv::msg(query)]
    fn admins(&self, ctx: QueryCtx) -> Result<AdminsResponse, Self::Error> {
        let admins: Vec<Addr> = self
            .admins
            .keys(ctx.deps.storage, None, None, cosmwasm_std::Order::Ascending)
            .collect::<Result<_, _>>()?;

        Ok(AdminsResponse { admins })
    }
}
```

We have something new here. First, `contract` has an attribute `module`. Its purpose is to tell
the macro where our contract is defined. It's required for pathing. You can expand the macro 
and check where it is used.

Next, there is the attribute mentioned before - `messages`. Its purpose is similar to the `module` 
with the difference that it provides ^sylvia with path to the interface. We also have to provide 
the interface's name, although it should be optional in the future.
We need to do one more thing, which is to add the `messages` attribute to the contract definition.

```rust,noplayground
#use cosmwasm_std::{Addr, DepsMut, Empty, Response, StdResult};
#use cw_storage_plus::{Item, Map};
#use sylvia::types::{ExecCtx, InstantiateCtx, QueryCtx};
#use sylvia::{contract, entry_points};
#
#use crate::error::ContractError;
#use crate::responses::CountResponse;
#
#pub struct CounterContract<'a> {
#    pub(crate) count: Item<u32>,
#    pub(crate) admins: Map<'static, &'a Addr, Empty>,
#}

#[entry_points]
#[contract]
#[sv::error(ContractError)]
#[sv::messages(crate::whitelist as Whitelist)]
impl CounterContract<'_> {
#   pub const fn new() -> Self {
#        Self {
#            count: Item::new("count"),
#            admins: Map::new("admins"),
#        }
#    }
#
#    #[sv::msg(instantiate)]
#    pub fn instantiate(&self, ctx: InstantiateCtx, count: u32) -> StdResult<Response> {
#        self.count.save(ctx.deps.storage, &count)?;
#        self.admins
#            .save(ctx.deps.storage, &ctx.info.sender, &Empty {})?;
#        Ok(Response::default())
#    }
#
#    #[sv::msg(query)]
#    pub fn count(&self, ctx: QueryCtx) -> StdResult<CountResponse> {
#        let count = self.count.load(ctx.deps.storage)?;
#        Ok(CountResponse { count })
#    }
#
#    #[sv::msg(exec)]
#    pub fn increment_count(&self, ctx: ExecCtx) -> Result<Response, ContractError> {
#        if !self.is_admin(&ctx.deps, &ctx.info.sender) {
#            return Err(ContractError::Unathorized(ctx.info.sender));
#        }
#        self.count
#            .update(ctx.deps.storage, |count| -> StdResult<u32> {
#                Ok(count + 1)
#            })?;
#        Ok(Response::default())
#    }
#
#    #[sv::msg(exec)]
#    pub fn decrement_count(&self, ctx: ExecCtx) -> Result<Response, ContractError> {
#        if !self.is_admin(&ctx.deps, &ctx.info.sender) {
#            return Err(ContractError::Unathorized(ctx.info.sender));
#        }
#        let count = self.count.load(ctx.deps.storage)?;
#        if count == 0 {
#            return Err(ContractError::CannotDecrementCount);
#        }
#        self.count.save(ctx.deps.storage, &(count - 1))?;
#        Ok(Response::default())
#    }
#
#    fn is_admin(&self, deps: &DepsMut, addr: &Addr) -> bool {
#        self.admins.has(deps.storage, addr)
#    }
}
```

Time to test if the new functionality works and is part of our contract.
Here suggest splitting the tests semantically, but in this example, we will add those tests
to the same test file.

```rust,noplayground
#[test]
fn manage_admins() {
    let app = App::default();
    let code_id = CodeId::store_code(&app);

    let owner = "owner".into_addr();
    let admin = "admin";

    let contract = code_id.instantiate(1).call(&owner).unwrap();

    // Admins list is empty
    let admins = contract.whitelist_proxy().admins().unwrap().admins;
    assert!(admins.is_empty());

    // Admin can be added
    contract
        .whitelist_proxy()
        .add_admin(admin.to_owned())
        .call(&owner)
        .unwrap();

    let admins = contract.whitelist_proxy().admins().unwrap().admins;
    assert_eq!(admins, &[Addr::unchecked(admin)]);

    // Admin can be removed
    contract
        .whitelist_proxy()
        .remove_admin(admin.to_owned())
        .call(&owner)
        .unwrap();

    let admins = contract.whitelist_proxy().admins().unwrap().admins;
    assert!(admins.is_empty());
}
```

We can add and remove admins. Now you can add the logic preventing users from incrementing and 
decrementing the count. You can extract the sender address by calling 
[`ctx.info.sender`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/struct.MessageInfo.html).
It would also be nice if the owner was an admin by default and if adding admins required the status 
of one.

# Next step

We have learned about almost all of the ^sylvia features. The next chapter will be about talking 
with remotes.
