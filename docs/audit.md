# Audit Findings

> Auto-managed by $x-review and $x-fix.

## Open

*(No open findings)*

---

## Won't Fix

*(none)*

> Rechecked after the ruc 11 / vsdb 17 / nix 0.32 / rand 0.10 bump. No open or deferred findings.

## Resolved — Overhaul (2026-11-16)

### [HIGH] tendermint: start treated one live process as a partial beacon failure
- **Where**: `src/tendermint_based/dev.rs` and `src/tendermint_based/ddev/mod.rs` `Node::start`
- **What**: The beacon rule "more than 2 processes means healthy, otherwise restart" stopped a Tendermint node that correctly has a single process, then started another.
- **Resolution**: Any positive process count refuses the start. Beacon keeps the three-process rule.

### [MEDIUM] build: `ruc::cmd::exec_output` is denied
- **Where**: beacon and tendermint `dev` / `ddev`
- **What**: `#![deny(warnings)]` rejects the deprecated `exec_output`. The crate did not compile.
- **Resolution**: Call `cmd::exec`. ruc 11.0.3 no longer has `exec_output`. Also drop the extra parentheses on `Box<dyn RucError>` in the Tendermint port error type.

Protected envs, `fork` child exit, port caches, and the unset `$CHAIN_DEV_EGG_REPO` were re-checked and left unchanged.

---
