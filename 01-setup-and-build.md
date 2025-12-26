# Setup and build

Anchor analogy: replaces `anchor init`/`anchor build` with standard Rust and Solana tooling.

## Dependencies

```toml
[dependencies]
pinocchio = { version = "0.10.0", features = ["cpi"] }
solana-address = "2.0"
solana-account-view = "1.0"
solana-program-error = "3.0"

pinocchio-system = "0.4.0"
pinocchio-token-2022 = "0.1.0"
```

### Feature-gated entrypoint

```toml
[features]
bpf-entrypoint = []
```

## Build

```bash
cargo build-sbf --features bpf-entrypoint
```
