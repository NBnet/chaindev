# Changelog

Current release: **0.54.4**. Older notes below 0.50 were incomplete and described defaults the code no longer has. Use `git log` for the commit history.

## 0.54.4

- Tendermint `start` no longer restarts a node that already has one live process. That check was copied from the beacon stack, which expects three processes.
- Command execution uses `ruc::cmd::exec`. ruc 11 removed `exec_output`, so the old call no longer compiles under `deny(warnings)`.

## 0.54

- Beacon genesis clones `$CHAIN_DEV_EGG_REPO` and fails when it is unset. There is no built-in clone URL.
- Env data lives under `/tmp/__CHAIN_DEV__/<beacon_based|tendermint_based>/...`, or under `$RUNTIME_CHAIN_DEV_BASE_DIR` when that is set. Local mode uses `__DEV__`. Distributed mode uses `__D_DEV__`. See the READMEs under `src/beacon_based/` and `src/tendermint_based/`.
- New envs are protected. Beacon `Destroy` and `KickNodes`, and Tendermint `Destroy` and `KickNode`, require `Unprotect` or `force`. Tendermint local `DestroyAll` does too. Tendermint distributed `DestroyAll` skips a protected env and still returns success when `force` is set. Beacon has no `DestroyAll`.
- The crate is edition 2024 and is built with the current stable toolchain. The Tendermint stack remains optional and less maintained.
