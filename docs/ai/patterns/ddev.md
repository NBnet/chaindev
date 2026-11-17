# Distributed Cluster Review Patterns

## Files
- `src/beacon_based/ddev/mod.rs` — multi-host beacon env, vsdb port set
- `src/beacon_based/ddev/remote.rs` — copy and edit files on nodes
- `src/tendermint_based/ddev/mod.rs` — multi-host Tendermint twin
- `src/tendermint_based/ddev/remote.rs`

## Architecture
- Not the same `Op` surface as local mode. Beacon ddev adds `MigrateNodes`,
  `PushHosts`, `KickHosts`, and the `Host*` ops, and still has no
  `DestroyAll`. Tendermint ddev adds `MigrateNode`, `PushHosts`,
  `KickHost`, and the `Host*` ops. Node homes and processes are placed
  on records from the host map.
- `GLOBAL_BASE_DIR` is `<BASE_DIR>/__D_DEV__`, so local and distributed
  port caches do not share a file.
- The port set is a vsdb map. Updates go through `unsafe { port_set.shadow() }`.
- Remote file ops (`chg_file_on_nodes`, `collect_files_from_nodes`,
  `collect_tgz_from_nodes`) build a per-node command and collect errors.

## Critical Invariants

### INV-DD1: Same Protection Rules as Local Mode
**Check**: `destroy` and kick refuse a protected env without `force` on
both stacks. Beacon has no `destroy_all`. Tendermint local `destroy_all`
honors `force`. Tendermint ddev `destroy_all` skips a protected env and
ignores `force`, then returns `Ok(())`.

### INV-DD2: Shadow Updates Match the Parent Set
**Check**: Every `shadow()` insert or remove has the matching update on
the vsdb map the rest of the function reads. A remove of a port the node
still holds is a bug. A failed remote start must not leave the port
allocated on the wrong host.

### INV-DD3: One Host Failure Fails the Op
**Check**: Remote helpers push errors into a list and finish through
`check_errlist!` (or an equivalent early `Err`). They do not return
`Ok(())` after a logged SSH failure.

### INV-DD4: Beacon EGG Rules Still Apply
**Check**: Distributed beacon genesis does not grow a default clone URL.
Tendermint ddev does not start cloning EGG just because beacon does.

## Common Bug Patterns

### Safety Fix on One Stack (technical-patterns.md 6.1)
`ddev` beacon got the port-release fix and Tendermint `ddev` did not.

### Shadow Drift (technical-patterns.md 3.1)
`shadow().remove` runs and the persistent set still contains the port, or
the reverse.

## Review Checklist
- [ ] `__D_DEV__` base did not collapse onto `__DEV__`
- [ ] Host pick uses `connection_addr_x` when a network id is in play
- [ ] File-change ops quote or reject remote paths the tool did not create
- [ ] Existing `#[allow(clippy::type_complexity)]` was not copied elsewhere
