# 📡 api

Shared `.proto` definitions and generated gRPC stubs for [CryptOS-PKI](https://github.com/CryptOS-PKI).

Published as a standalone, versioned Go module. Consumed by [`cryptos`](https://github.com/CryptOS-PKI/cryptos) and [`manager`](https://github.com/CryptOS-PKI/manager) as a Go dependency; consumed by [`web`](https://github.com/CryptOS-PKI/web) via generated TypeScript stubs.

## 📂 Layout

```
proto/cryptos/v1/    # .proto sources (the contract)
go/cryptos/v1/       # generated Go stubs (committed; no toolchain required to consume)
buf.yaml             # buf module + lint config (STANDARD)
buf.gen.yaml         # buf generation: protoc-gen-go + protoc-gen-go-grpc
Taskfile.yml         # fmt / lint / generate / test / ci targets
```

## 🧱 Phase 1 protos

| File | Defines |
|---|---|
| `node.proto` | `NodeService` — the 5 Phase 1 RPCs: `ApplyConfig`, `GetStatus`, `GetIdentity`, `StartCeremony` (server-streaming), `SignCSR` (debug-build-only). |
| `identity.proto` | `Identity` — DER + PEM + leaf SHA-256 for the CA chain. |
| `ceremony.proto` | `CeremonyEvent` stream messages + ceremony kind/event enums. |
| `status.proto` | `NodeStatus` — role, identity state, TPM state, etcd state, boot count. |
| `config.proto` | `MachineConfig` Phase 1 subset (role/network/storage/bootstrap/pki). |
| `audit.proto` | `AuditEvent` — hash-chained audit log entry shape. |

Phase 2 will add role-aware service splits, protocol-adapter management, and full machine-config schema. Phase 3 adds HA, fleet, extensions, and recovery RPCs.

## 🚀 Consuming

```bash
go get github.com/CryptOS-PKI/api@latest
```

```go
import cryptosv1 "github.com/CryptOS-PKI/api/go/cryptos/v1"
```

Generated TS stubs (for `web/`) ship in a follow-up release once Phase 2 frontend work begins.

## 🛠️ Contributing

Requires Go 1.24+, [`buf`](https://buf.build/docs/installation), `protoc-gen-go`, `protoc-gen-go-grpc`, and [`go-task`](https://taskfile.dev).

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

task ci          # fmt + lint + generate-and-verify + test
task generate    # regenerate Go stubs after a .proto change
task license     # re-inject Apache 2.0 headers via golic
```

The generated stubs under `go/cryptos/v1/` are committed; CI fails if they drift from the protos.

## 🚦 Status

**Pre-alpha.** The surface is unstable while Phase 1 lands. Tag-based semver kicks in once the v1 contract solidifies.

## 📄 License

[Apache License 2.0](LICENSE). Copyright 2026 Shane.
