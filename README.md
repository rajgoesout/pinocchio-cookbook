# Pinocchio Cookbook

Quick reference for Anchor developers switching to Pinocchio.

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `#[program]` | `entrypoint!(process_instruction)` |
| `#[derive(Accounts)]` | Manual validation / `TryFrom` |
| `#[account]` | `#[repr(C)]` + zero-copy |
| `Context<T>` | `(&Address, &[AccountView], &[u8])` |
| `Pubkey` | `Address` |
| `AccountInfo` | `AccountView` |

## Cargo.toml

```toml
[package]
name = "my-program"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[features]
bpf-entrypoint = []
cpi = ["pinocchio/cpi"]

[dependencies]
pinocchio = "0.9"
pinocchio-system = "0.3"
pinocchio-token = "0.4"
```

## Minimal Program

```rust
#![no_std]

use pinocchio::{AccountView, Address, ProgramResult, entrypoint, error::ProgramError};

pub const ID: Address = pinocchio::address!("YourProgramId11111111111111111111111111111");

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Address,
    accounts: &[AccountView],
    instruction_data: &[u8],
) -> ProgramResult {
    let [account, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    // Your logic here
    Ok(())
}
```

## Build

```bash
cargo build-sbf
```

## Docs

| Topic | Description |
|-------|-------------|
| [Accounts](./accounts.md) | AccountView API |
| [Programs](./programs.md) | Entrypoints & structure |
| [Instructions](./instructions.md) | Parsing instruction data |
| [State](./state.md) | Zero-copy state structs |
| [PDAs](./pda.md) | Program derived addresses |
| [CPI](./cpi.md) | Cross-program invocation |
| [Tokens](./tokens.md) | SPL Token operations |
| [Errors](./errors.md) | Custom errors |
| [Testing](./testing.md) | LiteSVM & Mollusk |

## Examples

- [Counter](./examples/counter.md) - Simple state with PDA
- [Vault](./examples/vault.md) - Token custody with PDA signing
