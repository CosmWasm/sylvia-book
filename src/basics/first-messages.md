# Generating first messages

We have set up our dependencies. Now let's use them to create simple messages.

## Creating an instantiation message

For this step we will create a new file:

- `src/contract.rs` - here, we will define our messages and behavior of the contract upon receiving
  them

Add this module to `src/lib.rs`. You want it to be public, as users might want to get access to
types stored inside your contract.

```rust,noplayground
pub mod contract;
```

Now let's create an `instantiate` method for our contract. In `src/contract.rs`

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use sylvia::contract;
use sylvia::types::InstantiateCtx;

pub struct CounterContract;

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

So what is going on here? First, we define the CounterContract struct. It is empty right now but 
later when we learn about states, we will use its fields to store them.
We mark the `impl` block with [`contract`](https://docs.rs/sylvia/0.7.0/sylvia/attr.contract.html)
attribute macro. It will parse every method inside the `impl` block marked with the `[msg(...)]` 
attribute and create proper messages and utilities like `multitest helpers` for them.
More on them later.

`CosmWasm` contract requires only the `instantiate` `entry_point`, and it is mandatory to specify
it for the `contract` macro. We have to provide it with the proper context type 
[`InstantiateCtx`](https://docs.rs/sylvia/0.7.0/sylvia/types/struct.InstantiateCtx.html).
Context gives us access to the blockchain state, information about our contract, and the sender of the 
message. We return the [`StdResult`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/type.StdResult.html)
which uses standard `CosmWasm` error 
[`StdError`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/enum.StdError.html).
It's generic over [`Response`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/struct.Response.html).
For now, we will return the `default` value of it.

I recommend expanding the macro now and seeing what `sylvia` generates. It might be overwhelming 
as there will be a lot of things generated that seem not relevant to our code, so for the bare 
minimum check the `InstantiateMsg` and its impl block.

## Next step
If we will build our contract with `cargo build --release --target wasm32-unknown-unknown --lib`
and run the `cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm`, it will fail with

```
contract & cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm
Available capabilities: {"cosmwasm_1_2", "iterator", "staking", "stargate", "cosmwasm_1_1", "cosmwasm_1_3"}

target/wasm32-unknown-unknown/release/contract.wasm: failure
Error during static Wasm validation: Wasm contract doesn't have required export: "instantiate". Exports required by VM: ["allocate", "deallocate", "instantiate"].

Passes: 0, failures: 1
```

This is because our contract is not yet complete. We defined the message that could be sent to it but
didn't provide any `entry_point`. In the next chapter, we will finally make it the proper contract.
