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

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, _ctx: InstantiateCtx) -> StdResult<Response> {
        Ok(Response::default())
    }
}
```

So what is going on here? First, we define the CounterContract struct. It is empty right now but 
later when we learn about states, we will use its fields to store them.
We mark the `impl` block with [`contract`](https://docs.rs/sylvia/latest/sylvia/attr.contract.html)
attribute macro. It will parse every method inside the `impl` block marked with the `[sv::msg(...)]` 
attribute and create proper messages and utilities like `multitest helpers` for them.
More on them later.

`CosmWasm` contract requires only the `instantiate` entry point, and it is mandatory to specify
it for the `contract` macro. We have to provide it with the proper context type 
[`InstantiateCtx`](https://docs.rs/sylvia/latest/sylvia/types/struct.InstantiateCtx.html).

Context gives us access to the blockchain state, information about our contract, and the sender of the 
message. We return the [`StdResult`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/type.StdResult.html)
which uses standard `CosmWasm` error 
[`StdError`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/enum.StdError.html).
It's generic over [`Response`](https://docs.rs/cosmwasm-std/1.3.1/cosmwasm_std/struct.Response.html).
For now, we will return the `default` value of it.

I recommend expanding the macro now and seeing what ^Sylvia generates.
It might be overwhelming, as there will be a lot of things generated that seem not relevant to our code,
so for the bare minimum check the `InstantiateMsg` and its `impl` block.

## Next step

If we build our contract with command:

```shell
contract $ cargo build --release --target wasm32-unknown-unknown --lib
```

and then run:

```shell
contract $ cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm
```
