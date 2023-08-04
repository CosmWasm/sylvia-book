# Entry points

Typical Rust application starts with the `fn main()` function called by the operating system.
Smart contracts are not significantly different. When the message is sent to the contract, a
function called "entry point" is called. Unlike native applications, which have only a single
`main` entry point, smart contracts have a couple corresponding to different message types:
`instantiate`, `execute`, `query`, `sudo`, `migrate` and more.

To start, we will go with three basic entry points:

- `instantiate` is called once per smart contract lifetime - you can think about it as
  a constructor or initializer of a contract.
- `execute` for handling messages which can modify contract state - they are used to
  perform some actual actions.
- `query` for handling messages requesting some information from a contract; unlike `execute`,
  they can never affect any contract state, and are used just like database queries.

## Generate entry points

`Sylvia` provides us with [`entry_points`](https://docs.rs/sylvia/0.7.0/sylvia/attr.entry_points.html)
attribute macro. In most cases, your entry point will just dispatch received messages to the handler,
so it's not necessary to manually create them, and we can rely on a macro to do that for us.

Let's add the attribute macro to our contract:

```
use cosmwasm_std::{Response, StdResult};
use sylvia::types::InstantiateCtx;
use sylvia::{contract, entry_points};

pub struct CounterContract;

#[entry_points]
#[contract]
impl CounterContract {
    pub const fn new() -> Self {
        Self
    }

    #[msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}
```

Note that `#[entry_points]` is added above the `#[contract]`. This is because `contract` removes
the attributes such as `#[msg(...)]` on which both those macros rely. Remember to always place 
`#[entry_points]` first.

`Sylvia` generates the entry_points with [`#[entry_point]`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/attr.entry_point.html)
attribute macro. Its purpose is to wrap the whole entry point to the form Wasm runtime understands. 
The proper Wasm entry points can use only basic types supported natively by Wasm specification, and 
Rust structures and enums are not in this set. Working with such entry points would be 
overcomplicated, so CosmWasm creators delivered the `entry_point` macro. It creates the raw Wasm 
entry point, calling the decorated function internally and doing all the magic required to build our 
high-level Rust arguments from arguments passed by Wasm runtime.

Now that our contract has a proper `entry point` let's build it and check if it's properly defined.

```
contract $ cargo build --release --target wasm32-unknown-unknown --lib
    Finished release [optimized] target(s) in 0.03s

contract $ cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm
Available capabilities: {"stargate", "cosmwasm_1_3", "cosmwasm_1_1", "cosmwasm_1_2", "staking", "iterator"}

target/wasm32-unknown-unknown/release/contract.wasm: pass

All contracts (1) passed checks!
```

## Next step

Nice! We have a proper `CosmWasm` contract. Now let's add some state to it so it will actually be 
able to do something.
