# Local Cluster Review Patterns

## Files
- `src/beacon_based/dev.rs` — local beacon `Env`, `Node`, `Op`, port cache
- `src/beacon_based/common.rs` — `NodePorts`, beacon `BASE_DIR`
- `src/tendermint_based/dev.rs` — local Tendermint twin
- `src/tendermint_based/common.rs` — Tendermint `NodePorts`, `BASE_DIR`
- `src/lib.rs` — `check_errlist!`

## Architecture
- An env home is `<GLOBAL_BASE_DIR>/envs/<name>`, with `CONFIG` beside the
  node homes. `GLOBAL_BASE_DIR` is `<BASE_DIR>/__DEV__`.
- Beacon `Op` is `Create`, `Destroy`, `PushNodes`, `KickNodes`, `Protect`,
  `Unprotect`, `Start`, `Stop`, `Restart`, `DebugFailedNodes`, `List`,
  `Custom`, and `Nil`. There is no `DestroyAll`.
- Tendermint `Op` adds `DestroyAll`, `PushNode`, `KickNode`, `StartAll`,
  `StopAll`, `Show`, and `ShowAll`. Local `DestroyAll` returns an error
  for a protected env unless `force` is set.
- Beacon genesis can be built by cloning `$CHAIN_DEV_EGG_REPO`.
  Tendermint genesis is generated in-process.
- Node start double-forks through `unistd::fork` and `exec_spawn`
  (`bash -c`).

## Critical Invariants

### INV-DEV1: Created Protected
`Env::create` sets `is_protected: true` before nodes are started.
**Check**: Both stacks. `Unprotect` is a separate persisted op.

### INV-DEV2: Destroy and Kick Honor `force`
**Check**: Both stacks. Beacon's variant is `KickNodes` and the node
method is `clean_up`. Tendermint's variant is `KickNode` and the node
method is `clean`. Fuhrers stay.

### INV-DEV3: Child of `fork` Exits
**Check**: `Node::start` in both files. Parent does not execute the child
command. `unsafe` covers only `fork`.

### INV-DEV4: Ports Are Reserved, Cached, and Released
**Check**: `alloc_ports` skips reserved and in-use ports. `PortsCache`
lives at `<GLOBAL_BASE_DIR>/ports_cache` (a file in local mode).
Beacon `Node::clean_up` and Tendermint `Node::clean` remove every port
in `get_port_list()` and then delete the home.

### INV-DEV5: EGG URL Comes From the Environment
**Check**: Beacon `gen_genesis` errors when `CHAIN_DEV_EGG_REPO` is unset.
Tendermint has no EGG clone.

## Common Bug Patterns

### Protection Bypass (technical-patterns.md 1.1)
The `is_protected` check moves below `stop` or `remove_dir_all`.

### Fork Fall-Through (technical-patterns.md 2.1)
The child arm returns `Ok(())` instead of `exit(0)`.

## Review Checklist
- [ ] Twin file in the other stack compared for the same safety rule
- [ ] `force_create` calls `destroy(true)` before deleting the home
- [ ] Beacon process-count threshold was not copied onto Tendermint
- [ ] `write_cfg` still runs after protect, unprotect, and kick
