---
name: x-review
description: Review a chaindev diff, commit range, or the full codebase for protection bypasses, process lifecycle, port allocation, and remote-exec defects, updating docs/audit.md only. Use only when the user explicitly invokes $x-review, or as the review phase of $x-overhaul.
---

# chaindev Deep Regression Review

Resolve repository paths from the Git root; Markdown links are relative to
this file. Read invocation arguments from the user prompt.

High-signal review. Code is read-only; `docs/audit.md` is the only write
allowed. A standalone review ends with findings and registry changes; it
does not commit, push, or deploy, and does not authorize code repairs.
Use [x-fix](../x-fix/SKILL.md) to resolve recorded findings,
[x-commit](../x-commit/SKILL.md) for self-reviewing commits, and
[x-overhaul](../x-overhaul/SKILL.md) for the full review-fix-commit pipeline.

## Setup

Read first:
[technical-patterns.md](../../../docs/ai/technical-patterns.md),
[review-core.md](../../../docs/ai/review-core.md), and
[false-positive-guide.md](../../../docs/ai/false-positive-guide.md).
For each affected subsystem, map changed files to the table below and read
the matching guide under [patterns/](../../../docs/ai/patterns/).

| Path | Subsystem | Guide |
|------|-----------|-------|
| `src/beacon_based/dev.rs`, `src/beacon_based/common.rs`, `src/tendermint_based/dev.rs`, `src/tendermint_based/common.rs`, `src/lib.rs` | local clusters | `dev.md` |
| `src/beacon_based/ddev/`, `src/tendermint_based/ddev/` | distributed clusters | `ddev.md` |
| `src/common/` | hosts and SSH | `remote.md` |

When a change lands in one stack, open the twin file in the other stack
before deciding the finding is local.

## Input

Accept at most one review scope supplied in the user's request:

| Input | Scope | How |
|-------|-------|-----|
| *(empty)* | Latest commit | `git diff HEAD~1`, `git log -1` |
| `N` (integer) | Last N commits | `git diff HEAD~N`, `git log -N --oneline` |
| `all` | Full codebase audit | Full Audit Protocol below |
| `<commit hash>` | Specific commit | `git diff <hash>~1 <hash>` |
| `<hash1>..<hash2>` | Commit range | `git diff <hash1> <hash2>` |

For diff-based scopes, proceed to the Execution Protocol. For `all`, use the
Full Audit Protocol. Reject anything else; never guess a scope.

## Execution Protocol

### 1. Context & Classification

1. Record a worktree baseline (`git status --short`, `git rev-parse HEAD`).
2. Read the full diff; identify ALL affected subsystems via the table above.
3. Read the corresponding pattern guide for each affected subsystem.
4. Classify each change per the review-core methodology (protection,
   destruction, spawn, ports, remote, genesis, stack drift).

### 2. Deep Regression Analysis

For each HIGH or CRITICAL classified change:

- Read at least 50 lines of surrounding context.
- Trace callers of the changed `Op` arm or helper.
- Compare the beacon and Tendermint twins when the rule is shared.
- Boundary conditions: protected plus `force` false, `KickNode` of a
  missing id, `CHAIN_DEV_EGG_REPO` unset, process count 0/1/2/>2, same
  vs different `local_network_id`.
- Failure paths: `fork` error, one host in a multi-host loop, `write_cfg`
  failure after an in-memory protect or kick.

Cross-reference every candidate with `technical-patterns.md` and try to
refute it with `false-positive-guide.md`; keep only concrete evidence.

### 3. Cross-Cutting Analysis

- Crash window: between `fork` and the child `exit`, and between home
  creation and `CONFIG` write.
- Feature split: default features omit `tendermint_based`. A shared-code
  change is not reviewed until both features are considered.
- API compatibility: `Op`, `EnvOpts`, `NodePorts`, `NodeCmdGenerator`.
  New trait methods need a default.

### 4. Code Style Enforcement

- No new `#[allow(...)]`. The existing
  `#[allow(clippy::type_complexity)]` in `src/beacon_based/ddev/mod.rs`
  stays where it is.
- New `unsafe` needs a `SAFETY` comment. Existing sites are the two
  `fork` calls and the `port_set.shadow()` calls.
- Doc-code alignment: `AGENTS.md`, root `README.md`,
  `src/beacon_based/README.md`, `src/tendermint_based/README.md`,
  `docs/ai/review-core.md`, this skill's full-audit table, and
  `docs/ai/patterns/`. Code paths under `/tmp/__CHAIN_DEV__/` win over
  the README layout when they disagree; report the drift when a change
  touches either side.

### 5. Unsafe Code Audit

Known `unsafe`:

- `unistd::fork` in `src/beacon_based/dev.rs` and `src/tendermint_based/dev.rs`
- `port_set.shadow()` in `src/beacon_based/ddev/mod.rs` and
  `src/tendermint_based/ddev/mod.rs`

For any added `unsafe` block: require a `SAFETY` comment, and check that
the child of `fork` cannot continue as a second manager.

### 6. Audit Registry (docs/audit.md)

1. Read `docs/audit.md` (create if absent).
2. **Prune**: verify each `## Open` entry against the current codebase;
   remove entries that are 100% fixed.
3. **Merge**: add new findings under `## Open`, deduplicating against
   existing entries, sorted CRITICAL → HIGH → MEDIUM → LOW.
4. **Preserve**: leave all `## Won't Fix` entries untouched.

File format:

```markdown
# Audit Findings

> Auto-managed by $x-review and $x-fix.

## Open

### [SEVERITY] subsystem: one-line summary
- **Where**: file:line_range
- **What**: description
- **Why**: invariant/pattern violated
- **Suggested fix**: how to fix

---

## Won't Fix

### [SEVERITY] subsystem: one-line summary
- **Where**: file:line_range
- **What**: description
- **Reason**: why this cannot or should not be fixed
```

## Output Format

```
## Review Summary

**Commit**: <hash> <subject>
**Subsystems**: <list>
**Risk Level**: CRITICAL / HIGH / MEDIUM / LOW

## Findings

### [SEVERITY] subsystem: one-line summary

**Where**: file:line_range
**What**: Description
**Why**: Invariant/pattern violated (cite technical-patterns.md)
**Fix**: Suggested fix

## No Issues Found

(list areas checked)
```

---

## Full Audit Protocol (`all` scope)

Audit all source files, not just uncommitted changes (~8.1K LOC). Partition
by subsystem; if delegation is available and authorized, launch one
read-only reviewer per group and keep ownership disjoint. The beacon and
Tendermint twins are separate groups so each is read in full.

| Subsystem | Paths | Pattern Guide |
|-----------|-------|---------------|
| local beacon | `src/beacon_based/dev.rs`, `src/beacon_based/common.rs` | `dev.md` |
| local tendermint | `src/tendermint_based/dev.rs`, `src/tendermint_based/common.rs` | `dev.md` |
| distributed beacon | `src/beacon_based/ddev/` | `ddev.md` |
| distributed tendermint | `src/tendermint_based/ddev/` | `ddev.md` |
| hosts and remote | `src/common/`, `src/lib.rs` | `remote.md` |

Each reviewer reads all source files in its group, loads the pattern guide,
and performs deep analysis (protection, destruction, ports, spawn, remote
errors). Aggregate, deduplicate, and specifically compare twins for shared
safety rules. Update `docs/audit.md` per the Audit Registry step.

Report:

```
## Full Audit Report

**Scope**: All sources (~8.1K LOC)
**Subsystems Audited**: <list>
**Total Findings**: N (X critical, Y high, Z medium, W low)

## Findings
(sorted by severity, grouped by subsystem)

## Clean Areas
(subsystems with no findings — list what was checked)
```
