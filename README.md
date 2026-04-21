# agent-rs-proto

Shared gRPC contract for the [agent-rs](https://github.com/cosming20/agent-rs) platform.

**This repository ships only `.proto` source files.** No Rust / TypeScript / Python generated code is checked in — every consumer compiles from the `.proto` files using its own toolchain (`tonic-build`, `connect-es`, `grpc-tools`, etc.).

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
