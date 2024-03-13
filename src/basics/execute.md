# Execution messages

We created `instantiate` and `query` messages. We have the state in our contract and can test it.
Now let's expand our contract by adding the possibility of updating the state. In this chapter, we
will add the `increase_count` execute message.

## Add a message

Adding a new variant of `ExecMsg` is as simple as adding one to the `QueryMsg`.

```rust,noplayground
use cosmwasm_std::{Response, StdResult};
use cw_storage_plus::Item;
use sylvia::types::{ExecCtx, InstantiateCtx, QueryCtx};
use sylvia::{contract, entry_points};

use crate::responses::CountResponse;

pub struct CounterContract {
    pub(crate) count: Item<'static, u32>,
}

#[entry_points]
#[contract]
impl CounterContract {
    pub const fn new() -> Self {
        Self {
            count: Item::new("count"),
        }
    }

    #[sv::msg(instantiate)]
    pub fn instantiate(&self, ctx: InstantiateCtx, count: u32) -> StdResult<Response> {
        self.count.save(ctx.deps.storage, &count)?;
        Ok(Response::default())
    }

    #[sv::msg(query)]
    pub fn count(&self, ctx: QueryCtx) -> StdResult<CountResponse> {
        let count = self.count.load(ctx.deps.storage)?;
        Ok(CountResponse { count })
    }

    #[sv::msg(exec)]
    pub fn increment_count(&self, ctx: ExecCtx, ) -> StdResult<Response> {
        self.count
            .update(ctx.deps.storage, |count| -> StdResult<u32> {
                Ok(count + 1)
            })?;
        Ok(Response::default())
    }
}
```

We will add the `#[sv::msg(exec)]` attribute and make it accept [`ExecCtx`](https://docs.rs/sylvia/latest/sylvia/types/struct.ExecCtx.html)
parameter. It will return `StdResult<Response>`, similiar to the instantiate method.
Inside we call [`update`](https://docs.rs/cw-storage-plus/1.1.0/cw_storage_plus/struct.Item.html#method.update)
to increment the `count` state.
Like that new variant for the `ExecMsg` is created, `execute` entry point properly dispatches a message
to this method and our multitest helpers are updated and ready to use.

Again I encourage you to expand the macro and inspect all three things mentioned above.

## Testing

Our contract has a new variant for the `ExecMsg`. Let's check if it works properly.

```rust,noplayground
use sylvia::multitest::App;

use crate::contract::mt::CodeId;

#[test]
fn instantiate() {
    let app = App::default();
    let code_id = CodeId::store_code(&app);

    let owner = "owner";

    let contract = code_id.instantiate(42).call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 42);

    contract.increment_count().call(owner).unwrap();

    let count = contract.count().unwrap().count;
    assert_eq!(count, 43);
}
```

As in the case of query we can call `increment_count` directly on our proxy contract. Same as in 
the case of instantiate we have to pass the caller here. We could also send funds here using `with_funds` 
method.

## Next step

I encourage you to add a new `ExecMsg` variant by yourself. It could be called `set_count`. Test it 
and see how easy it is to add new message variants even without the guide.
