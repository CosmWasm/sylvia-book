# Override entry point

It may happen that for any reason CosmWasm will start support some new
entry point that is not yet implemented in ^sylvia. There is a way to
add it manually using `#[sv::override_entry_point(...)]` attribute.
This feature can be used to override already implemented entry points
like `execute` and `query`.

## Example

To make ^sylvia generate multitest helpers with `custom_entrypoint` support, you first need to define your
`entry point`.

```rust,noplayground
#[entry_point]
pub fn custom_entrypoint(deps: DepsMut, _env: Env, _msg: SudoMsg) -> StdResult<Response> {
    CounterContract::new().counter.save(deps.storage, &3)?;
    Ok(Response::new())
}
```

You have to define the `CustomEntrypointMsg` yourself, as it is not yet supported.

```rust,noplayground
#[cfg_attr(not(feature = "library"), entry_points)]
#[contract]
#[sv::override_entry_point(custom=crate::entry_points::custom_entrypoint(crate::messages::CustomEntrypointMsg))]
impl CounterContract {
}
```

For every `entry point,` provide the path to the function in a separate attribute. You also have to
provide the type of your custom msg, as multitest helpers need to deserialize an array of bytes.

# Next step

In the next chapter, we will learn about custom messages.
