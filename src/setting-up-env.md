# Setting up the environment

To work with CosmWasm smart contract, you will need rust installed on your
machine. If you don't have one, you can find installation instructions on [the
Rust website](https://www.rust-lang.org/tools/install).

I assume you are working with a stable Rust channel in this book.

Additionally, you will need the Wasm rust compiler backend installed to build
Wasm binaries. To install it, run:

```
rustup target add wasm32-unknown-unknown
```

## Check contract utility

An additional helpful tool for building smart contracts is the `cosmwasm-check`
utility. It allows you to check if the wasm binary is a proper smart contract
ready to upload into the blockchain. You can install it using cargo:

```
$ cargo install cosmwasm-check
```

If the installation succeeds, you should be able to execute the utility from your command line.

```
$ cosmwasm-check --version
Contract checking 1.2.1
```

## Verifying the installation

To guarantee you are ready to build your smart contracts, you need to make sure you can build examples.
Checkout the [sylvia](https://github.com/CosmWasm/sylvia) repository and run the testing command in
its folder:

```
$ git clone git@github.com:CosmWasm/sylvia.git
$ cd ./sylvia
sylvia $ cargo test
```

You should see that everything in the repository gets compiled, and all tests pass. 

`sylvia` framework contains some examples of contracts. To find them go to `examples/contracts` directory.
These contracts are maintained by CosmWasm creators, so contracts in there should follow good practices.

To verify the `cosmwasm-check` utility, first, you need to build a smart contract. Go to some contract
directory, for example, `examples/contracts/cw1-whitelist`, and call `cargo wasm`:

```
sylvia $ cd examples/contracts/cw1-whitelist
sylvia/examples/contracts/cw1-whitelist $ cargo wasm
```

`wasm` is an alias for `"build --release --target wasm32-unknown-unknown --lib"`.
You should be able to find your output binary in the `examples/target/wasm32-unknown-unknown/release/`
of the root repo directory - not in the contract directory itself! Now you can check if contract
validation passes:

```
sylvia $ cosmwasm-check examples/target/wasm32-unknown-unknown/release/cw1_whitelist.wasm

Available capabilities: {"cosmwasm_1_2", "cosmwasm_1_3", "staking", "iterator", "stargate", "cosmwasm_1_1"}

examples/target/wasm32-unknown-unknown/release/cw1_whitelist.wasm: pass

All contracts (1) passed checks!
```

## Macro expansion

Sylvia generates a lot of code for us which is not visible in code. To see what code is generated
with it go to `examples/contracts/cw1-whitelist/src/contract.rs`. In VSCode you can click on
`#[contract]`, do `shift+p` and then type: `rust analyzer: Expand macro recursively`. This will open
a window with fully expanded macro which you can browse. This is also possible f.e. in VIM depending
on your configuration. You can also use `cargo expand` tool from CLI for this.
