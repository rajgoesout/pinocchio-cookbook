# Why Pinocchio

Anchor makes Solana development feel high-level; Pinocchio keeps the program code efficient and dependency-free. The tradeoff is that you do the wiring yourself.

## Anchor concept → Pinocchio equivalent

- `#[program]` → `entrypoint!(process_instruction)`
- `#[derive(Accounts)]` → manual account validation
- `#[account]` → `#[repr(C)]` structs + zero-copy casts
- `#[error_code]` → custom enum + `ProgramError::Custom`
- `emit!` → `sol_log_data` wrapper
- `anchor_spl::token` → `pinocchio-token-2022`

Goal: you keep the same program behavior, but control layout, parsing, and compute costs directly.
