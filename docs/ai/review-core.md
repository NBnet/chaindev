# chaindev Review Core Methodology

This document defines the systematic review protocol for chaindev changes.

---

## Phase 1: Context Gathering

Before analyzing any change, gather context:

1. **Read the diff.**
2. **Identify affected subsystems**:
   - `src/beacon_based/dev.rs`, `src/beacon_based/common.rs` → local beacon
   - `src/tendermint_based/dev.rs`, `src/tendermint_based/common.rs` →
     local Tendermint
   - `src/beacon_based/ddev/` → distributed beacon
   - `src/tendermint_based/ddev/` → distributed Tendermint
   - `src/common/` → host records and SSH
   - `src/lib.rs` → `check_errlist!` and re-exports
3. **Load** the matching file under `docs/ai/patterns/`.
4. **Check the twin stack.** A safety change in beacon `dev` or `ddev`
   must be compared with the Tendermint file of the same role, and the
   reverse. They are copied structures, not one generic.
5. **Check both feature sets** when the diff touches `src/common/` or
   shared control flow. Default features omit `tendermint_based`.

## Phase 2: Change Classification

| Category | Description | Risk Level |
|----------|-------------|------------|
| Protection | `is_protected`, `force`, `Unprotect` | CRITICAL |
| Destruction | `destroy`, `destroy_all`, `kick_node`, `force_create`, `clean` | CRITICAL |
| Process spawn | `fork`, start/stop command lines | CRITICAL |
| Port cache | allocate, reserve, remove | HIGH |
| Remote execution | SSH command and file copy | HIGH |
| Genesis / EGG | clone URL, premine, genesis apply | HIGH |
| Host addressing | local IP vs external IP, network id | HIGH |
| Stack drift | beacon vs Tendermint safety rules | HIGH |
| Error aggregation | `check_errlist!`, per-host errors | MEDIUM |
| Config persistence | `CONFIG` write, serde defaults | MEDIUM |
| Documentation | README path or feature drift | LOW |

## Phase 3: Regression Analysis

For each HIGH or CRITICAL change:

### 3.1 Invariant Check

**Protection.** New envs set `is_protected: true`. `destroy`, `destroy_all`,
and `kick_node` return an error when the env is protected and `force` is
false. `protect` / `unprotect` persist through `write_cfg`. `force_create`
may destroy an existing env, and that call passes `force = true` on purpose.

**Destruction ordering.** `destroy` stops processes, cleans each node
(ports released, home removed), then removes the env home. A failure to
stop is logged and cleanup still proceeds only where the existing code
already uses `info_omit!` on stop. Do not skip port release.

**Ports.** Reserved ports are refused. The cache file is
`<GLOBAL_BASE_DIR>/ports_cache`. A port taken by a node is recorded before
the process starts and removed in `clean`. `ddev` updates the vsdb port set
through `shadow()`; the parent map and the shadow must not disagree.

**Process spawn.** `fork` is `unsafe`. The child writes a log line, runs
`exec_spawn`, and `exit`s. The parent returns `Ok(())` without running
child-only code. A process count above the healthy threshold refuses to
start a second copy. Beacon's healthy count is 3 (execution, beacon,
validator). Do not apply that threshold to a Tendermint node that is one
process.

**Remote.** `HostAddr::connection_addr` prefers `ext_ip`.
`connection_addr_x` uses `local_ip` only when `local_network_id` matches.
Host commands go through `Remote::exec_cmd`. Per-host failures are collected
by `check_errlist!`; an empty error list is `Ok(())`.

**Genesis.** Beacon genesis clones `$CHAIN_DEV_EGG_REPO` and errors when
the variable is unset. No default URL.

**Homes.** Beacon `BASE_DIR` is
`/tmp/__CHAIN_DEV__/beacon_based/<suffix>/<host>+<user>` unless
`RUNTIME_CHAIN_DEV_BASE_DIR` is set. Tendermint uses
`/tmp/__CHAIN_DEV__/tendermint_based/<suffix>/<host>/<user>`. Local mode
appends `/__DEV__`; distributed mode appends `/__D_DEV__`. The stack
READMEs describe this layout. Report a doc change that would send an
operator to a different tree, including any revival of the old
`/__chain_dev__/` paths.

### 3.2 Boundary Conditions

- Protected env, `force = false` and `force = true`.
- `DestroyAll` when one env is protected and another is not.
- `KickNode` with `None` (last ordinary node) and with an unknown id.
- `force_create` over an existing home.
- Zero initial validators; only a fuhrer.
- Port already in the cache; port in the reserved list; cache file missing.
- Same `local_network_id` vs different ids; `ext_ip` absent.
- `$CHAIN_DEV_EGG_REPO` unset; repo URL containing shell metacharacters.
- Beacon process count 0, 1, 2, and greater than 2.
- `RUNTIME_CHAIN_DEV_BASE_DIR` set and unset.

### 3.3 Failure Paths

- `fork` returns `Err`.
- Child `exec_spawn` fails: the child must exit rather than continue as a
  second management process.
- Remote command fails on one host of many: the aggregated `Result` is
  `Err`.
- `write_cfg` fails after an in-memory `protect` or `kick`: the on-disk
  `CONFIG` must not claim a state the disk does not have. Today these
  paths write after the memory change and return the write error; do not
  make the memory change stick without a successful write, and do not
  delete data before the stop/clean result is known.

### 3.4 Concurrency

Port allocation and env-config writes are process-local plus a file or a
vsdb map. Report a race only when two operations in one process can hand
out the same port, or when `shadow()` drops an update the parent map kept.
Do not require a cross-process lock that the tool has never had, unless the
change claims one.

## Phase 4: Cross-Cutting Concerns

### 4.1 Crash Window

`kill -9` after `fork` and before the child `exit`, or after a home is
created and before `CONFIG` is written, can leave a running node or an
orphan directory. A change to start/destroy should name that window. Port
cache entries must still be removable by a later `clean`.

### 4.2 Stack Drift

Quote both files when a protection, port, or destroy change lands on only
one of beacon and Tendermint. A beacon-only genesis or client-count change
is not drift.

### 4.3 API Contract

`Op` variants, `EnvOpts` fields, and the `NodeCmdGenerator` /
`NodePorts` traits are public. New trait methods need defaults or every
caller breaks. Feature defaults stay `beacon_based`.

### 4.4 Code Style

Violations are LOW:
- No new `#[allow(...)]`.
- New `unsafe` needs a `SAFETY` comment.
- Doc-code alignment: `AGENTS.md`, the stack READMEs, `README.md`,
  `docs/ai/review-core.md`, `.agents/skills/x-review/SKILL.md`, and
  `docs/ai/patterns/`.

## Phase 5: Reporting

```
[SEVERITY] subsystem: one-line summary

WHERE: file:line_range
WHAT: Description of the issue
WHY: Invariant or pattern from technical-patterns.md
FIX: Suggested fix, or the question that blocks one
```

- **CRITICAL**: protected env destroyed without force, fork child continues
  as a manager, shell metacharacter in a management command
- **HIGH**: port double-booked, wrong host, genesis from an implicit URL,
  beacon/Tendermint safety drift
- **MEDIUM**: error dropped, config not written, partial cleanup
- **LOW**: docs, clarity
- **INFO**: observation, not a bug

Only report findings with concrete evidence. Consult
[false-positive-guide.md](false-positive-guide.md) before finalizing.
