# Remote type

Your contract may rely on communication with another one. For example, it could
`instantiate` a `CW20` contract and, during the workflow, send `Mint` messages to it. If `CW20` 
contract was created using `sylvia`, it would have a `Remote` type generated which would make this
process more user friendly.
Currently, it is only possible to send queries using `Remote` but support for the `execute` messages
is on the way.

To check some examples, checkout the [`sylvia`](https://github.com/CosmWasm/sylvia) repository
and go to `sylvia/tests/remote.rs`.

## Working with Remote

`Remote` represents some contract instantiated on the blockchain. It aims to give contract
developers a gateway to communicate with other contracts. It has only one field, which is a remote
contract address.
It exposes only a single method called `querier`, which returns a `BoundQuerier` type.
`BoundQuerier` has a method for every contract/interface query.
If we create a contract relying on our `CounterContract`, it could query its state as below.

```rust,noplayground
let count = Remote::new(addr)
    .querier(&ctx.deps.querier)
    .count()?
    .count;

let admins = crate::whitelist::Remote::new(ctx.info.sender)
    .querier(&ctx.deps.querier)
    .admins()?;
```

In case of contract initializing the `CW20` contract you might want to keep its address in the
state.

```rust,noplayground
struct Contract {
    cw20: cw20::Remote<'static>,
}
```

Then somewhere in your code just create the querier from your remote and query.

```rust,noplayground
self.cw20
    .querier(&ctx.deps.querier)
    .query_all_balances()?
```

# Next step

In front of us is the last chapter of `sylvia` basics. Let's learn about some good practices before
we will get to writing our own contracts.