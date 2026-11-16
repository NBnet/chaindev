[![Rust](https://github.com/rust-util-collections/chaindev/actions/workflows/rust.yml/badge.svg)](https://github.com/rust-util-collections/chaindev/actions/workflows/rust.yml)
[![Latest Version](https://img.shields.io/crates/v/chaindev.svg)](https://crates.io/crates/chaindev)
[![Rust Documentation](https://img.shields.io/badge/api-rustdoc-blue.svg)](https://docs.rs/chaindev)
![GitHub top language](https://img.shields.io/github/languages/top/rust-util-collections/chaindev)
[![Minimum rustc version](https://img.shields.io/badge/rustc-1.85+-lightgray.svg)](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0.html)

# chaindev

Powerful development and testing utils for blockchain developers.

- [Beacon(ETH2) based chains](src/beacon_based)
    - `feature = [ "beacon_based" ]`, default
- [Tendermint/CometBFT based chains](src/tendermint_based)
    - `feature = [ "tendermint_based" ]`
    - **WARNING**: less maintained, new features may be missing!

### ENV VARs

- `$CHAIN_DEV_EGG_REPO`: git URL cloned when a beacon genesis is built from scratch. Required for that path. There is no default URL.
- `$RUNTIME_CHAIN_DEV_BASE_DIR`: overrides the per-user data root described in the stack READMEs.

Homes, port caches, and the beacon/Tendermint split are documented in [src/beacon_based/README.md](src/beacon_based/README.md) and [src/tendermint_based/README.md](src/tendermint_based/README.md).
