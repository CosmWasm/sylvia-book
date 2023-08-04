# Building the contract

Now that our contract correctly compiles into wasm, let's digest our build command.

```
$ cargo build --target wasm32-unknown-unknown --release --lib
```

The `--target` argument tells cargo to perform cross-compilation for a given target instead of
building a native binary for an OS, it is running on - in this case, `wasm32-unknown-unknown`,
which is a fancy name for Wasm target.

Contract would be properly created without `--lib`, but later, when we add `query`, it will be 
needed so adding this argument from the beginning is a good idea.

Additionally, I passed the `--release` argument to the command - it is not
required, but in most cases, debug information is not very useful while running
on-chain. It is crucial to reduce the uploaded binary size for gas cost
minimization. It is worth knowing that there is a [CosmWasm Rust
Optimizer](https://github.com/CosmWasm/rust-optimizer) tool that takes care of
building even smaller binaries. For production, all the contracts should be compiled using this
tool, but it is not an essential thing to do for learning purposes.

## Aliasing build command

Now I see you are disappointed in building your contracts with some overcomplicated command
instead of simple `cargo build`. Hopefully, it is not the case. The common practice is to alias
the building command to make it as simple as building a native app.

Let's create the `.cargo/config` file in your contract project directory with the following content:

```toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release --lib"
wasm-debug = "build --target wasm32-unknown-unknown --lib"
```

Building your Wasm binary is as easy as executing `cargo wasm`. We also added the additional
`wasm-debug` command for rare cases when we want to build the wasm binary, including debug information.

## Checking contract validity

When the contract is built, the last step is to ensure it is a valid CosmWasm contract is to call
`cosmwasm-check` on it:

```
$ cargo wasm
...
$ cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm
Available capabilities: {"cosmwasm_1_1", "iterator", "staking", "stargate"}

target/wasm32-unknown-unknown/release/contract.wasm: pass

All contracts (1) passed checks!
```
