# Sudo entry point

Sylvia supports a `sudo` type entry point both in interfaces and in
contracts. Those methods can be used as a part of the network's
governance procedures. More informations can be found in official
CosmWasm documentation. From ^sylvia user point of view there's no
much difference between `sudo` and `exec` methods.

## Example

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use sylvia::types::{InstantiateCtx, SudoCtx};
use sylvia::{contract, entry_points};


pub mod interface {
    use cosmwasm_std::{Response, StdResult, StdError};
    use sylvia::types::{SudoCtx};
    use sylvia::interface;

    #[interface]
    pub trait Interface {
        type Error: From<StdError>;

        #[sv::msg(sudo)]
        fn interface_sudo_msg(&self, _ctx: SudoCtx) -> StdResult<Response>;
    }
}

pub struct CounterContract;

#[entry_points]
#[contract]
#[sv::messages(interface)]
impl CounterContract {
    pub const fn new() -> Self {
        Self
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::default())
    }

    #[sv::msg(sudo)]
    pub fn sudo_method(&self, _ctx: SudoCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}

impl interface::Interface for CounterContract {
    fn interface_sudo_msg(&self, _ctx: SudoCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}
```