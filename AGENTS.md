# Working on chaindev

chaindev manages blockchain dev clusters. The beacon (ETH2) stack is the
default and the one that is kept current. The Tendermint/CometBFT stack is
still compiled behind `tendermint_based` and is less maintained. Each stack
has a local multi-process mode (`dev`) and a multi-host mode (`ddev`).

This is a dev-environment tool, not a consensus engine. The safety
properties are: protected envs stay protected, destructive ops do not run
by accident, port caches do not double-book, remote commands stay bound to
the intended host, and the beacon and Tendermint stacks do not silently
diverge on those rules.

## AI workflows

This file is the shared agent entrypoint. Reusable workflows live in
`.agents/skills/` and require explicit invocation; ordinary requests do
not implicitly authorize commits:

- `$x-review`: deep regression review of a diff, N commits, a hash/range, or
  `all`. Updates [docs/audit.md](docs/audit.md); does not commit.
- `$x-fix`: resolve the `docs/audit.md` backlog, self-review, and commit.
- `$x-commit`: self-review uncommitted changes, fix, format, bump the patch
  version, and commit.
- `$x-overhaul`: full codebase audit → fix → commit pipeline.

Shared guides live under [docs/ai/](docs/ai/):
`technical-patterns.md` (cataloged cluster-management bug patterns),
`review-core.md` (systematic review methodology),
`false-positive-guide.md` (rules for filtering spurious findings), and
`patterns/` (local dev, distributed dev, hosts/remote).

## Layout

```
src/lib.rs                         # crate root, check_errlist!
src/common/                        # EnvName, CustomOps, hosts, SSH remote
src/beacon_based/dev.rs            # local beacon envs
src/beacon_based/ddev/             # multi-host beacon envs
src/beacon_based/common.rs         # ports, BASE_DIR
src/tendermint_based/dev.rs        # local Tendermint envs
src/tendermint_based/ddev/         # multi-host Tendermint envs
src/tendermint_based/common.rs
```

About 8.1K lines of Rust. `beacon_based` and `tendermint_based` are
parallel implementations, not a shared generic with two configs.

## Build & test

```bash
make fmt          # cargo fmt
make lint         # cargo clippy --all-features, then again with --tests
make test         # cargo test, then cargo test --release; both use default features and --test-threads=1
```

The crate root is `#![deny(warnings)]`. Default features are
`beacon_based`. A change that touches `src/common/` or
`src/tendermint_based/` is not covered by `make test` alone; also run
`cargo test --all-features -- --test-threads=1`. `make lint` already
passes `--all-features`.

## Conventions

- No new `#[allow(...)]`. One existing
  `#[allow(clippy::type_complexity)]` in `src/beacon_based/ddev/mod.rs` is
  not a license to add another.
- `unsafe` is limited to `unistd::fork` in both `dev.rs` files and
  `port_set.shadow()` in both `ddev` modules. A new `unsafe` block needs
  a `SAFETY` comment.
- New envs are created `is_protected: true`. Beacon public ops are
  `Destroy` and `KickNodes` (no `DestroyAll`). Tendermint public ops are
  `Destroy`, `KickNode`, and `DestroyAll`. Those refuse a protected env
  unless `force` is set, except distributed Tendermint `DestroyAll`, which
  skips a protected env and still returns success.
- `$CHAIN_DEV_EGG_REPO` has no default. Do not reintroduce a hardcoded
  clone URL.
- Public behavior changes must update the matching README under
  `src/beacon_based/` or `src/tendermint_based/` and the guide under
  `docs/ai/patterns/`.
- Keep repository documentation, instructions, and commit messages in
  English. Respond to the user in their language.
