# Beacon (ETH2) cluster management

Local (`dev`) and multi-host (`ddev`) helpers for beacon clusters. This is the default feature (`beacon_based`).

## Data root

Unless `RUNTIME_CHAIN_DEV_BASE_DIR` is set, the root is:

```text
/tmp/__CHAIN_DEV__/beacon_based/<suffix>/<hostname>+<user>
```

`<suffix>` is the compile-time value of `STATIC_CHAIN_DEV_BASE_DIR_SUFFIX`, or empty. The hostname and the user name are joined with `+`.

| Mode | Home of env `<name>` | Port cache |
|------|----------------------|------------|
| local `dev` | `<root>/__DEV__/envs/<name>` | `<root>/__DEV__/ports_cache` |
| distributed `ddev` | `<root>/__D_DEV__/envs/<name>` | `<root>/__D_DEV__/ports_cache` |

Each env is created protected. `Destroy`, `DestroyAll`, and `KickNode` fail until `Unprotect`, unless the call passes `force`.

## Genesis

When the env has no genesis yet, `gen_genesis` clones `$CHAIN_DEV_EGG_REPO` and builds from that checkout. The variable has no default. A hardcoded clone URL is not used.
