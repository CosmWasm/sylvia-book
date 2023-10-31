# Building the contract

Now that our contract correctly compiles into Wasm, let's digest our build command.

```shell
$ cargo build --target wasm32-unknown-unknown --release --lib
```

The `--target` argument tells cargo to perform cross-compilation for a given target, instead of
building a native binary for an OS it is running on. In this case the target is `wasm32-unknown-unknown`,
which is a fancy name for Wasm target.

Our contract would be also properly compiled without `--lib` flag, but later, when we add `query`,
this flag will be required, so using it from the beginning is a good habit.

Additionally, I passed the `--release` argument to the command - it is not
required, but in most cases, debug information is not very useful while running
on-chain. It is crucial to reduce the uploaded binary size for gas cost
minimization. It is worth knowing that there is a [CosmWasm Rust
Optimizer](https://github.com/CosmWasm/rust-optimizer) tool that takes care of
building even smaller binaries. For production, all the contracts should be compiled using this
tool, but it is now not essential for learning purposes.

## Aliasing build command

Now I see you are disappointed in building your contracts with some overcomplicated command
instead of simple `cargo build`. Hopefully, it is not the case. The common practice is to alias
the building command, to make it as simple as building a native application.

Let's create `.cargo/config` file in your contract project directory with the following content:

```toml
[alias]
wasm = "build --target wasm32-unknown-unknown --release --lib"
wasm-debug = "build --target wasm32-unknown-unknown --lib"
```

Building your Wasm binary is now as easy as executing `cargo wasm`. We also added the additional
`wasm-debug` command for rare cases, when we want to build the Wasm binary with debug information included.

## Checking contract validity

When the contract is built, the last step to ensure that it is a valid CosmWasm contract
is to call `cosmwasm-check` on it:

```shell
$ cargo wasm
    .
    . (compilation messages)
    .
    Finished release [optimized] target(s) in 0.03s

$ cosmwasm-check target/wasm32-unknown-unknown/release/contract.wasm
Available capabilities: {"cosmwasm_1_1", "iterator", "staking", "stargate"}

target/wasm32-unknown-unknown/release/contract.wasm: pass

All contracts (1) passed checks!
```
