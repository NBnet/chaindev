# Changelog

Current release: **0.54.3**. Older notes below 0.50 were incomplete and described defaults the code no longer has. Use `git log` for the commit history.

## 0.54

- Beacon genesis clones `$CHAIN_DEV_EGG_REPO` and fails when it is unset. There is no built-in clone URL.
- Env data lives under `/tmp/__CHAIN_DEV__/<beacon_based|tendermint_based>/...`, or under `$RUNTIME_CHAIN_DEV_BASE_DIR` when that is set. Local mode uses `__DEV__`. Distributed mode uses `__D_DEV__`. See the READMEs under `src/beacon_based/` and `src/tendermint_based/`.
- New envs are protected. `Destroy`, `DestroyAll`, and `KickNode` require `Unprotect` or `force`.
- The crate is edition 2024 (Rust 1.85 or newer). The Tendermint stack remains optional and less maintained.
