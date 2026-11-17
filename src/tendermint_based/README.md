# Tendermint cluster management

Local (`dev`) and multi-host (`ddev`) helpers for Tendermint/CometBFT clusters. Enable with `--features tendermint_based`. This stack is less maintained than `beacon_based`. New beacon features are not automatically mirrored here.

The optional Rust dependency is `tendermint` 0.41.0 (plus `tendermint-config` 0.41.0). Do not assume a particular `tendermint-core` git branch from this crate alone.

Protection, port release, and destroy ordering follow the same rules as the beacon stack. Genesis is generated in-process. This stack does not clone `$CHAIN_DEV_EGG_REPO`.

## Data root

Unless `RUNTIME_CHAIN_DEV_BASE_DIR` is set, the root is:

```text
/tmp/__CHAIN_DEV__/tendermint_based/<suffix>/<hostname>/<user>
```

`<suffix>` is the compile-time value of `STATIC_CHAIN_DEV_BASE_DIR_SUFFIX`, or empty. Unlike the beacon root, the hostname and the user name are separate path components.

| Mode | Home of env `<name>` | Port cache |
|------|----------------------|------------|
| local `dev` | `<root>/__DEV__/envs/<name>` | `<root>/__DEV__/ports_cache` |
| distributed `ddev` | `<root>/__D_DEV__/envs/<name>` | `<root>/__D_DEV__/ports_cache` |

Each env is created protected. `Destroy` and `KickNode` fail until `Unprotect`, unless the call passes `force`. Local `DestroyAll` does the same. Distributed `DestroyAll` skips a protected env and returns success even when `force` is set.
