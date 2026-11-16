# chaindev False Positive Guide

Before reporting any finding, check it against this guide.

---

## FP-1: Rust Ownership Already Prevents It

**Pattern**: Reporting memory unsafety in safe Rust.
**Rule**: `unsafe` exists only at the `unistd::fork` sites in both
`dev.rs` files and the `port_set.shadow()` sites in both `ddev` modules.
Report memory unsafety only inside those blocks, or for a newly added
`unsafe`.

## FP-2: Tendermint Stack Is Behind Beacon on Purpose

**Pattern**: Reporting that `tendermint_based` lacks a beacon-only feature
(EGG genesis, execution/consensus client split, premine addresses).
**Rule**: The Tendermint module is documented as less maintained. Report a
divergence only when a shared safety rule differs: protection, force
flags, port-cache updates, destroy ordering, or host addressing.

## FP-3: Protected Env Plus `force` Is the Override

**Pattern**: Reporting that `Destroy { force: true }` deletes a protected
env.
**Rule**: `force` is the explicit override. Report it only when `force` is
ignored, when a protected env is destroyed with `force == false`, or when
`force_create` destroys an existing env without going through `destroy`.

## FP-4: Fuhrer Nodes Are Not Kicked

**Pattern**: Reporting that `KickNode` cannot remove a fuhrer.
**Rule**: Fuhrers are kept by design. `KickNode` only removes entries from
`meta.nodes`. Report it only if a fuhrer is removed, or if kicking the last
ordinary node panics instead of returning an error.

## FP-5: Partial Process Count Restarts the Node

**Pattern**: Reporting that fewer than 3 live processes triggers `stop`
then start.
**Rule**: Beacon nodes expect execution, beacon, and validator processes.
A count of 1 or 2 is treated as a partial failure and restarted. Report
it only if a healthy single-process Tendermint node is judged by the
beacon threshold, or if the restart stop uses `force` in a way that skips
a required cleanup.

## FP-6: SSH Output and Remote Paths Are Dev-Machine Data

**Pattern**: Reporting that a dev cluster reads remote files or runs a
management command over SSH.
**Rule**: That is the product. Report it only when the command is built
from an unsanitized field that can escape the intended argv, when the
wrong host is selected (`connection_addr` vs `connection_addr_x`), or when
a failure on one host is dropped and the overall `Result` is still `Ok`.

## FP-7: `$CHAIN_DEV_EGG_REPO` Must Be Set

**Pattern**: Reporting that beacon genesis fails when the env var is unset.
**Rule**: There is intentionally no default URL. Report a hardcoded clone
URL, or a path that clones when the variable is missing.

## FP-8: Unwrap/Expect on Known-Valid State

**Pattern**: Reporting `unwrap()` / `pnk!` as a panic.
**Rule**: `pnk!` on the child side of `fork` aborts the child, which is
the intended failure mode for a failed spawn. Report a `pnk!` or `unwrap`
on the parent side that can kill the management process for a bad env
name, a missing port, or a remote error that should be a `Result`.

## FP-9: Clippy or `deny(warnings)` Would Catch It

**Pattern**: Reporting formatting or unused imports.
**Rule**: `make lint` runs clippy for all features, including tests. The
crate root denies warnings. Focus on semantic bugs.

## FP-10: "Consider" Without a Concrete Downside

**Pattern**: Advisory suggestions with no trigger.
**Rule**: Every finding needs a concrete operation (`Destroy`, `KickNode`,
`Create`, migrate, remote exec) and a concrete bad outcome (protected data
deleted, port reused, genesis cloned from the wrong URL, node left
running).

## FP-11: Test-Only or Example Strings

**Pattern**: Holding unit tests to production process rules.
**Rule**: This crate has little test-only process management. Report a
test only when it locks in the wrong protection or port behavior.

## FP-12: Base Directory Under `/tmp` Is Intentional

**Pattern**: Reporting that env homes live under `/tmp/__CHAIN_DEV__/`.
**Rule**: `BASE_DIR` is per host and per user, overridable with
`RUNTIME_CHAIN_DEV_BASE_DIR`. Report it only if two users or two stacks
(beacon vs Tendermint, `dev` vs `ddev`) resolve to the same directory, or
if the README path is updated without the code (or the reverse) and a
caller would look in the wrong place.
