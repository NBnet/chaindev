[![Rust](https://github.com/NBnet/chaindev/actions/workflows/rust.yml/badge.svg)](https://github.com/NBnet/chaindev/actions/workflows/rust.yml)
[![Latest Version](https://img.shields.io/crates/v/chaindev.svg)](https://crates.io/crates/chaindev)
[![Rust Documentation](https://img.shields.io/badge/api-rustdoc-blue.svg)](https://docs.rs/chaindev)
![GitHub top language](https://img.shields.io/github/languages/top/NBnet/chaindev)

# chaindev

Powerful development and testing utils for blockchain developers.

- [Beacon(ETH2) based chains](src/beacon_based)
    - Cargo feature `beacon_based`, the default
- [Tendermint/CometBFT based chains](src/tendermint_based)
    - Cargo feature `tendermint_based` (`--features tendermint_based`)
    - **WARNING**: less maintained, new features may be missing!

### ENV VARs

- `$CHAIN_DEV_EGG_REPO`: git URL cloned when a beacon genesis is built from scratch. Required for that path. There is no default URL.
- `$RUNTIME_CHAIN_DEV_BASE_DIR`: overrides the per-user data root described in the stack READMEs.

Homes, port caches, and the beacon/Tendermint split are documented in [src/beacon_based/README.md](src/beacon_based/README.md) and [src/tendermint_based/README.md](src/tendermint_based/README.md).
