# agent-rs-proto

Shared gRPC contract for the [agent-rs](https://github.com/cosming20/agent-rs) platform.

**This repository ships only `.proto` source files.** No Rust / TypeScript / Python generated code is checked in — every consumer compiles from the `.proto` files using its own toolchain (`tonic-build`, `connect-es`, `grpc-tools`, etc.).

## Architecture (2026-04-21 stateless pivot)

Agent-rs is fully stateless about conversations. All chat state lives in
agent-rs-web's Postgres and is replayed inline with every Ask request.
Documents are addressed end-to-end by their MinIO object key; indexing is
decoupled from Ask via a RabbitMQ queue consumed by the IndexerWorker.

**Surface:**

| RPC | Shape | Purpose |
|------|-------|---------|
| `Ask` | server-streaming | Replay history + docs → stream tool calls / partial answers / final |
| `EnqueueIndex` | unary | Publish a newly-uploaded MinIO key to the indexing queue |
| `GetDocumentStatus` | unary | Poll indexing state machine (pending/indexing/complete/failed) |
| `DeleteDocument` | unary | Purge Qdrant + Neo4j + index-state row for a key |

The web app uploads bytes DIRECTLY to MinIO (its own S3 credentials) and
only invokes this gRPC surface for control-plane coordination.


## Layout

```
proto/
  agent.proto       # the Agent service + messages, package agent.v1
buf.yaml            # `buf lint` / `buf breaking` config
```

## Consumers

| Repo | Language | Build step |
|------|----------|------------|
| [agent-rs](https://github.com/cosming20/agent-rs) | Rust (tonic server) | `tonic-build` in `crates/agent-service/build.rs` |
| [agent-rs-web](https://github.com/cosming20/agent-rs-web) | Rust (Leptos SSR + axum + tonic client) | `tonic-build` in `build.rs` |

## Adding this repo as a submodule (recommended)

From the consumer's repo root:

```bash
git submodule add https://github.com/cosming20/agent-rs-proto.git proto/agent-rs-proto
```

Then in `build.rs`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .build_server(true)   // or false for client-only consumers
        .build_client(true)
        .compile_protos(
            &["proto/agent-rs-proto/proto/agent.proto"],
            &["proto/agent-rs-proto/proto"],
        )?;
    Ok(())
}
```

## Wire-compat rules

- Fields may only be **added**; renumbering, retyping, or removing a field is a breaking change.
- New RPCs append to the `Agent` service — never insert in the middle.
- Breaking changes go to a new file `agent.v2.proto` with package `agent.v2`.

`buf breaking` (configured in `buf.yaml`) enforces the above against `main` on every PR.

## Local linting

```bash
brew install bufbuild/buf/buf
buf lint
buf breaking --against '.git#branch=main'
```

## License

MIT — same as `agent-rs`.
