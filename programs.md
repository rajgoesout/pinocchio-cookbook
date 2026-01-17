# Programs

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `#[program]` | `entrypoint!(process_instruction)` |
| `declare_id!` | `pub const ID: Address = address!("...")` |
| Automatic discriminator dispatch | Manual `match` on first byte |
| `Context<T>` | Account slice destructuring |

## Basic Structure

```rust
#![no_std]

use pinocchio::{AccountView, Address, ProgramResult, entrypoint, error::ProgramError};

pub const ID: Address = pinocchio::address!("MyProgram11111111111111111111111111111111");

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Address,
    accounts: &[AccountView],
    instruction_data: &[u8],
) -> ProgramResult {
    if program_id != &ID {
        return Err(ProgramError::IncorrectProgramId);
    }

    match instruction_data.first() {
        Some(&0) => initialize(accounts, &instruction_data[1..]),
        Some(&1) => update(accounts, &instruction_data[1..]),
        _ => Err(ProgramError::InvalidInstructionData),
    }
}

fn initialize(accounts: &[AccountView], data: &[u8]) -> ProgramResult { Ok(()) }
fn update(accounts: &[AccountView], data: &[u8]) -> ProgramResult { Ok(()) }
```

**Anchor equivalent:**
```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> { Ok(()) }
    pub fn update(ctx: Context<Update>) -> Result<()> { Ok(()) }
}
```

## File Layout

```
src/
├── lib.rs           # entrypoint + instruction dispatch
├── state.rs         # account state structs
├── error.rs         # custom errors
└── processor/
    ├── mod.rs
    ├── initialize.rs
    └── update.rs
```

## lib.rs with Modules

```rust
#![no_std]

mod state;
mod error;
mod processor;

use pinocchio::{AccountView, Address, ProgramResult, entrypoint, error::ProgramError};

pub const ID: Address = pinocchio::address!("...");

#[cfg(feature = "bpf-entrypoint")]
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Address,
    accounts: &[AccountView],
    instruction_data: &[u8],
) -> ProgramResult {
    processor::dispatch(program_id, accounts, instruction_data)
}
```

## Entrypoint Options

```rust
// Standard - includes allocator + panic handler
entrypoint!(process_instruction);

// Just entrypoint - you provide allocator/panic
program_entrypoint!(process_instruction);
pinocchio::default_allocator!();
pinocchio::default_panic_handler!();

// Lazy - parse accounts on-demand (saves CU for simple programs)
lazy_program_entrypoint!(process_instruction);
```

## Lazy Entrypoint

Parse accounts only when needed (useful for single-instruction programs):

```rust
use pinocchio::entrypoint::InstructionContext;

lazy_program_entrypoint!(process_instruction);
default_allocator!();
default_panic_handler!();

fn process_instruction(mut ctx: InstructionContext) -> ProgramResult {
    let first = ctx.next_account()?;   // Parse first account
    let second = ctx.next_account()?;  // Parse second account
    let data = ctx.instruction_data()?;
    let program_id = ctx.program_id()?;
    Ok(())
}
```

## No Allocator (Zero Heap)

```rust
#![no_std]
pinocchio::no_allocator!();
pinocchio::nostd_panic_handler!();
```

## Building

```bash
cargo build-sbf                    # Standard build
cargo build-sbf --features cpi     # With CPI support
```
