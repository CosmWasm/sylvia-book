# Override entry point

^Sylvia is still developing and lacks features like f.e. `sudo` support.
If you need to use a lacking feature of `CosmWasm` or prefer to define some custom
entry point, it is possible to use the `#[sv::override_entry_point(...)]` attribute.

## Example

To make ^sylvia generate multitest helpers with `sudo` support, you first need to define your
`entry point`.

```rust,noplayground
#[entry_point]
pub fn sudo(deps: DepsMut, _env: Env, _msg: SudoMsg) -> StdResult<Response> {
    CounterContract::new().counter.save(deps.storage, &3)?;
    Ok(Response::new())
}
```

You have to define the `SudoMsg` yourself, as it is not yet supported.

```rust,noplayground
#[cfg_attr(not(feature = "library"), entry_points)]
#[contract]
#[sv::override_entry_point(sudo=crate::entry_points::sudo(crate::messages::SudoMsg))]
#[sv::override_entry_point(exec=crate::entry_points::execute(crate::messages::CustomExecMsg))]
impl CounterContract {
}
```

For every `entry point,` provide the path to the function in a separate attribute. You also have to
provide the type of your custom msg, as multitest helpers need to deserialize an array of bytes.

# Next step

In the next chapter, we will learn about custom messages.
