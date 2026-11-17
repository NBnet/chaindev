# chaindev Technical Bug Patterns

Catalog of bug categories in this cluster manager. Load it before a review
or a fix.

---

## Category 1: Protection Bypass

### 1.1 Destructive Op Ignores `is_protected`

**Pattern**: `destroy`, `destroy_all`, or `kick_node` proceeds when
`is_protected` is true and `force` is false.
**Where**: both `src/beacon_based/dev.rs` and `src/tendermint_based/dev.rs`,
and the matching methods under both `ddev/` modules.
**Impact**: A default env, which is created protected, is deleted or loses
a node.
**Check**: The guard returns `Err` with the "unprotect it first" message
before `stop` or `clean`. Beacon has no `DestroyAll`. Tendermint local
`DestroyAll` passes `force` into each env. Tendermint ddev `DestroyAll`
skips a protected env and does not treat `force` as an override.
`protect` and `unprotect` call `write_cfg`.

### 1.2 `force_create` Deletes Without the Destroy Path

**Pattern**: `Create` with `force_create` removes the home directory without
stopping processes or releasing ports.
**Where**: `Env::create` in each stack.
**Impact**: Orphan processes and ports stuck in `ports_cache`.
**Check**: The existing env is loaded and `destroy(true)` runs before
`remove_dir_all`.

---

## Category 2: Process Lifecycle

### 2.1 Fork Child Falls Through

**Pattern**: The `ForkResult::Child` arm does not `exit` after `exec_spawn`.
**Where**: `Node::start` in `src/beacon_based/dev.rs` and
`src/tendermint_based/dev.rs`.
**Impact**: The child keeps running management code (further forks, config
writes) beside the parent.
**Check**: Child path is log, spawn, `exit(0)`. Parent path is `Ok(())`.
`fork` failure is `Err`, not a panic on the parent. The `unsafe` block
stays limited to the `fork` call.

### 2.2 Second Start Against a Live Node

**Pattern**: The process-count check is removed, or beacon's "more than 2
processes" threshold is applied to Tendermint.
**Where**: `Node::start`.
**Impact**: Two node supervisors for one home, or a healthy Tendermint
process treated as a partial beacon failure and killed.
**Check**: Beacon refuses when more than 2 processes match and restarts
when the count is 1 or 2. Tendermint must use its own command's notion of
"already running".

### 2.3 Stop Command Built From an Untrusted Fragment

**Pattern**: A node id, home, or extra opts value is interpolated into
`bash -c` without being a path the tool itself created.
**Where**: `NodeCmdGenerator::cmd_for_start`, `cmd_for_stop`,
`cmd_cnt_running`, and `exec_spawn`.
**Impact**: `app_extra_opts` is an intentional pass-through for the app
under test. A newly added field from a remote or a file is not.
**Check**: New interpolations are either tool-generated paths under the
env home or documented operator input. Do not wrap operator input in an
extra `sh -c`.

---

## Category 3: Ports

### 3.1 Port Handed Out Twice

**Pattern**: Allocation checks the OS bind but not `ports_cache`, or
removes a port before the process has exited.
**Where**: `alloc_ports`, `PortsCache`, `Node::clean`, and the `ddev`
`port_set.shadow()` updates.
**Impact**: Two nodes share a P2P or RPC port. The second fails later, or
both appear up.
**Check**: Reserved ports are excluded. The cache is updated before start
and removed in `clean` for every port in `get_port_list()`. A `shadow()`
remove that is not paired with the parent set is a bug.

### 3.2 Cache File Shared Across Stacks

**Pattern**: Beacon and Tendermint, or `dev` and `ddev`, resolve
`GLOBAL_BASE_DIR` to the same directory.
**Where**: `BASE_DIR` in each `common.rs`, `GLOBAL_BASE_DIR` in each `dev.rs`
and `ddev/mod.rs`.
**Impact**: One stack frees a port the other stack still uses.
**Check**: Beacon base contains `beacon_based` and `<host>+<user>`.
Tendermint base contains `tendermint_based` and `<host>/<user>`. Local
appends `__DEV__`; distributed appends `__D_DEV__`.

---

## Category 4: Remote Hosts

### 4.1 Wrong Address Selected

**Pattern**: `connection_addr_x` uses `ext_ip` for a host on the same
`local_network_id`, or uses `local_ip` across networks.
**Where**: `src/common/hosts.rs`.
**Impact**: Nodes advertise an unreachable address. Distributed clusters
fail to peer.
**Check**: Same network id selects `local_ip`. Otherwise `ext_ip` if set,
else `local_ip`.

### 4.2 Per-Host Error Dropped

**Pattern**: A loop over hosts ignores `exec_cmd` / file-copy errors.
**Where**: `src/common/remote.rs`, `src/*/ddev/remote.rs`.
**Impact**: A file or command is missing on one machine and the caller
proceeds to start the cluster.
**Check**: Failures accumulate and `check_errlist!` returns `Err` when the
list is non-empty.

---

## Category 5: Genesis and EGG

### 5.1 Implicit Clone URL

**Pattern**: Beacon genesis clones a hardcoded Git URL when
`$CHAIN_DEV_EGG_REPO` is unset.
**Where**: `gen_genesis` in `src/beacon_based/dev.rs` and
`src/beacon_based/ddev/mod.rs`.
**Impact**: The tool fetches and builds whatever that URL points at. The
default was removed for this reason.
**Check**: Unset variable is an error. The URL is used only for
`git clone` of the operator-supplied repo.

### 5.2 Genesis Applied to a Subset of Nodes

**Pattern**: `apply_genesis(None)` skips fuhrers or skips a node that was
just pushed.
**Where**: `create`, `push_node`, `apply_genesis`.
**Impact**: A new node starts without the env genesis, or a fuhrer has a
different chain id.
**Check**: `None` means every node. `push_node` applies genesis to the new
id and then starts that id.

---

## Category 6: Stack Drift

### 6.1 Safety Fix Applied to One Stack

**Pattern**: A protection, port, or destroy fix edits beacon only, or
Tendermint only.
**Where**: the four `dev` / `ddev` implementations.
**Impact**: The less-used stack keeps the bug. Callers who flip
`--features tendermint_based` hit it.
**Check**: Diff the twin file. Beacon-only EGG and client-count logic is
allowed to differ. Protection and port rules are not.
