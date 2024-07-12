# Attributes forwarding

This feature allows ^sylvia users to forward any attribute to any message
type using `#[sv::msg_attr(msg_type, ...)]` attribute.
For the messages that resolves to enum types it is possible to forward attributes to their specific variants by using `#[sv::attr(...)]` on top of the appropriate method - this works for `exec`, `query`
and `sudo` methods.

## Example

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use sylvia::types::{InstantiateCtx, ExecCtx};
use sylvia::{contract, entry_points};


pub mod interface {
    use cosmwasm_std::{Response, StdResult, StdError};
    use sylvia::types::QueryCtx;
    use sylvia::interface;

    #[interface]
    #[sv::msg_attr(query, derive(PartialOrd))]
    pub trait Interface {
        type Error: From<StdError>;

        #[sv::msg(query)]
        #[sv::attr(serde(rename(serialize = "QuErY")))]
        fn interface_query_msg(&self, _ctx: QueryCtx) -> StdResult<Response>;
    }
}

pub struct CounterContract;

#[entry_points]
#[contract]
#[sv::msg_attr(exec, derive(PartialOrd))]
impl CounterContract {
    pub const fn new() -> Self {
        Self
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::default())
    }

    #[sv::msg(exec)]
    #[sv::attr(serde(rename(serialize = "EXEC_METHOD")))]
    pub fn exec_method(&self, _ctx: ExecCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}
```