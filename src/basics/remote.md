# Remote type

Your contract may rely on communication with another one. For example, it could
`instantiate` a `CW20` contract and, during the workflow, send `Mint` messages to it. If `CW20` 
contract was created using ^sylvia, it would have a `Remote` type generated which would make this
process more user friendly.
Currently, it is only possible to send queries using `Remote` but support for the `execute` messages
is on the way.

To check some examples, checkout the ^sylvia repository
and go to `sylvia/tests/remote.rs`.

## Working with Remote

`Remote` represents some contract instantiated on the blockchain. It aims to give contract
developers a gateway to communicate with other contracts. It has only one field, which is a remote
contract address.
It exposes only a single method called `querier`, which returns a `BoundQuerier` type.
`BoundQuerier` has a method for every contract and interface query.
If we create a contract relying on our `CounterContract`, it could query its state as below.

```rust,noplayground
let count = Remote::<CounterContract>::new(addr)
    .querier(&ctx.deps.querier)
    .count()?
    .count;

let admins = crate::whitelist::Remote::<CounterContract>::new(ctx.info.sender)
    .querier(&ctx.deps.querier)
    .admins()?;
```

Important to note is that `Remote` is generic over the contract type. To use it in context
of some contract, just initialize it generic over it.

In case of contract initializing the `CW20` contract you might want to keep its address in the
state.

```rust,noplayground
use sylvia::types::Remote;

struct Contract<'a> {
    cw20: Item<Remote<'a, Cw20Contract>>,
}
```

Then to query the contract load the remote, call `querier` on it which will return `BoundQuerier`
and then call the query method on it.

```rust,noplayground
self.cw20
    .load(ctx.deps.storage)?
    .querier(&ctx.deps.querier)
    .query_all_balances()?
```

# Next step

Phew.. that was a journey. We learned most of the ^sylvia features and should be ready to create our first contracts.
In the last chapter, we will learn about some of the best practices that will make our code more readable and maintainable.
