# Hosts and Remote Review Patterns

## Files
- `src/common/hosts.rs` — `HostAddr`, host map, weight
- `src/common/remote.rs` — `Remote`, `exec_cmd`, put/get file, multi-host helpers
- `src/common/mod.rs` — `EnvName`, `CustomOps`, `NodeCmdGenerator`

## Architecture
- A host has `local_ip`, `local_network_id`, and an optional `ext_ip`.
  The display form is `local_ip` or `local_ip|ext_ip`, with an optional
  `<network-id>%` prefix on the local part.
- `Remote` wraps `ruc::ssh::RemoteHost` (user, port, local private key).
- Multi-host helpers run the same command or file operation on each host
  and join failures with newlines via `check_errlist!`.

## Critical Invariants

### INV-RM1: Address Selection
`connection_addr` is `ext_ip` if present, otherwise `local_ip`.
`connection_addr_x(id)` uses `local_ip` only when `local_network_id == id`.
**Check**: Parsing rejects more than one `|` and more than one `%`.
The default network id is the fixed placeholder
`default_mlm_mmm_lqm_qmzl_9527`, so two hosts that both omit the id are
treated as the same network.

### INV-RM2: Remote Errors Propagate
**Check**: `exec_cmd`, `put_file`, `get_file`, and the multi-host wrappers
return `Err` on SSH or command failure. A helper that starts nodes must
not ignore that `Err`.

### INV-RM3: Command Text Is the Operator Command
**Check**: New format strings that embed host fields use the parsed IP,
not the raw user string, once parsing has succeeded. Do not pass the raw
`HostAddr` display form back through a shell as a URL.

### INV-RM4: Trait Defaults Stay Safe
**Check**: `NodeCmdGenerator::cmd_for_migrate` defaults to
`Err("Unimplemented yet")`. `CustomOps` for `()` is a no-op. A new
required trait method without a default breaks every stack.

## Common Bug Patterns

### Wrong Address (technical-patterns.md 4.1)
Same-network peers are given `ext_ip` and cannot connect.

### Dropped Host Error (technical-patterns.md 4.2)
The loop `let _ = remote.exec_cmd(...)` and then returns `Ok(())`.

## Review Checklist
- [ ] Parse and display round-trip for `ip`, `id%ip`, and `ip|ext`
- [ ] Private key path is the host record's path, not a constant
- [ ] Weight / occupied-port probes that fail do not look like an empty host
