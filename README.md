# 📡 api

Shared `.proto` definitions and generated gRPC stubs for [CryptOS-PKI](https://github.com/CryptOS-PKI).

Published as a standalone, versioned Go module. Consumed by [`cryptos`](https://github.com/CryptOS-PKI/cryptos) and [`manager`](https://github.com/CryptOS-PKI/manager) as a Go dependency; consumed by [`web`](https://github.com/CryptOS-PKI/web) via generated TypeScript stubs.

## 📂 Layout

```
proto/cryptos/v1/        # node .proto sources (the contract)
proto/cryptos/fleet/v1/  # Fleet Manager .proto sources (FleetService)
go/cryptos/              # generated Go stubs (committed; no toolchain required to consume)
gen/ts/cryptos/          # generated TypeScript stubs for web/ (committed)
buf.yaml                 # buf module + lint config (STANDARD)
buf.gen.yaml             # buf generation: Go (protobuf, gRPC, Connect) + TypeScript (protoc-gen-es)
Taskfile.yml             # fmt / lint / generate / test / ci targets
```

## 🧱 Phase 1 protos

| File | Defines |
|---|---|
| `node.proto` | `NodeService` — the per-node management surface: config (`ApplyConfig`, `GetConfig`), status and identity, the first-boot ceremony, CA signing and revocation, fetching an issued certificate and its chain by serial (`GetIssuedCertificate`), key escrow and rotation, reset, in-place image upgrade, `Reboot` (orderly, CN-confirmed reboot or power-off), and SCEP administration (one-time enrolment challenges and the approval queue). |
| `identity.proto` | `Identity` — DER + PEM + leaf SHA-256 for the CA chain. |
| `ceremony.proto` | `CeremonyEvent` stream messages + ceremony kind/event enums. |
| `status.proto` | `NodeStatus` — role, identity state, TPM state, etcd state, boot count, the revocation preflight result (state, last error, when it was checked), the DNS resolver source and nameservers, each enrolment protocol's configured and running state, and whether a stored config change is waiting for a reboot. |
| `config.proto` | `MachineConfig` Phase 1 subset (role/network/storage/bootstrap/pki), plus the ACME, EST and SCEP enrolment blocks on `Pki` (`acme`, `est`, `scep`), each with an explicit `enabled` switch that takes effect at the next boot. |
| `scep.proto` | The SCEP challenge and approval-queue messages. A challenge is returned once, when it is minted, and no read carries it or its digest. |
| `audit.proto` | `AuditEvent` — hash-chained audit log entry shape. |
| `fleet/v1/fleet.proto` | `FleetService` — the Fleet Manager's surface over the fleet: node, certificate, profile, adapter, enrollment and operator-credential RPCs, the manager audit log (`ListAudit`), MCP agent key management (`ListMcpKeys`, `RevokeMcpKey`, `CreateMcpKey`), and step-up approvals (`ListApprovals`, `DecideApproval`). |

### Fetching an issued certificate

A client fetches an issued certificate and its chain by serial:

| RPC | Takes | Returns |
|---|---|---|
| `NodeService.GetIssuedCertificate` | `serial_hex` | `certificate_der`, `chain_der` (repeated, issuer up to root), `status`, `revoked_at` |
| `FleetService.GetCertificate` | `node_name`, `serial_hex` | `certificate_pem`, `chain_pem` (issuer up to root), `status`, `revoked_at` |

- `status` is `valid`, `revoked` or `expired`. `revoked_at` is RFC3339, empty unless the certificate is revoked.
- An unknown serial is `NotFound`. `GetCertificate` is readable at viewer level and above.

### MCP agent keys and audit actors

The Fleet Manager serves an MCP endpoint for AI agents. An agent authenticates with a long-lived bearer key bound to the serial of the operator certificate that minted it. `FleetService` carries the key management the web UI needs:

| RPC | Does |
|---|---|
| `ListMcpKeys` | Lists the caller's keys. An admin sets `all` to list every operator's keys. |
| `RevokeMcpKey` | Revokes a key by `id`. Operators revoke their own keys; admins revoke any. |
| `CreateMcpKey` | Mints a key with a `label` and an optional `level_ceiling`, for MCP clients without the OAuth login. The response's `plaintext_key` is returned once and never again. |

- `McpKey` holds `id`, `label`, `client_name`, `operator_cn`, `operator_serial`, `level_ceiling`, `created_at`, `last_used_at` and `revoked_at`. It never carries the key or its hash.
- A key is identity only. Its effective level is the lower of the bound certificate's live level and `level_ceiling` (`viewer`, `operator` or `admin`), and it stops working when that certificate is revoked or expires.
- These RPCs accept an operator certificate only; an MCP key can never mint, list or revoke keys.

`AuditEvent` (fleet) records the actor on every entry: `actor_kind` (`cert` or `mcp_key`), `actor_cn`, `actor_serial`, `key_id`, `via` (`web`, `mcp` or `api`), `tool`, `request_digest` and `outcome` (`ok`, `denied`, `pending` or `error`). `approval_id` and `approver_serial` name the step-up approval behind an entry and are empty when none applied. All actor fields are empty on entries recorded before the manager captured an actor.

### Step-up approvals

Some MCP tool calls need a human decision before they run. The manager records each one as an `Approval`, and the web UI lists and decides them:

| RPC | Does |
|---|---|
| `ListApprovals` | Lists approvals newest first. `status` filters to `pending`, `approved`, `denied`, `expired` or `used`; empty lists all. |
| `DecideApproval` | Approves (`approve` true) or denies a pending approval by `id` and returns the updated `Approval`. |

- `Approval` holds `id`, `tool`, `summary`, `request_digest` (lowercase hex SHA-256 of the canonical request), `requested_by_cn`, `requested_by_serial`, `key_id`, `required_level` (`viewer`, `operator` or `admin`), `created_at`, `expires_at`, `status`, `decided_by_cn`, `decided_by_serial` and `decided_at`. Timestamps are RFC3339 strings, empty when unset.
- Both RPCs accept an operator certificate only; an MCP key can never list or decide approvals. The decider's level must be at least `required_level`.

Phase 2 will add role-aware service splits, protocol-adapter management, and full machine-config schema. Phase 3 adds HA, fleet, extensions, and recovery RPCs.

## 🚀 Consuming

```bash
go get github.com/CryptOS-PKI/api@latest
```

```go
import cryptosv1 "github.com/CryptOS-PKI/api/go/cryptos/v1"
```

```go
import fleetv1 "github.com/CryptOS-PKI/api/go/cryptos/fleet/v1"
```

Generated TypeScript stubs for `web/` live under `gen/ts/cryptos/` (Connect-ES v2, `protoc-gen-es`).

## 🛠️ Contributing

Requires Go 1.25+, [`buf`](https://buf.build/docs/installation), `npx`, and [`go-task`](https://taskfile.dev).

The codegen plugins are pinned, so every machine generates the same bytes. `task generate` installs `protoc-gen-go`, `protoc-gen-go-grpc` and `protoc-gen-connect-go` into `.bin/` at the versions set in `Taskfile.yml`, built with the pinned Go toolchain, and `buf.gen.yaml` runs them from there. Any copies on your `PATH` are ignored. Run `task generate` rather than a bare `buf generate`. To bump a plugin, change its version in `Taskfile.yml` and commit the regenerated tree in the same PR.

```bash
task ci          # fmt + lint + generate-and-verify + test
task generate    # regenerate Go and TypeScript stubs after a .proto change
task tools       # install the pinned codegen plugins into .bin/ (generate runs this)
task license     # re-inject Apache 2.0 headers via golic
```

The generated stubs under `go/` and `gen/ts/` are committed; `task ci` fails if they drift from the protos. No GitHub workflow runs that check yet (#73), so run `task ci` before you push.

## 🚦 Status

**Pre-alpha.** The surface is unstable while Phase 1 lands. Tag-based semver kicks in once the v1 contract solidifies.

## 📄 License

[Apache License 2.0](LICENSE). Copyright 2026 Shane.
