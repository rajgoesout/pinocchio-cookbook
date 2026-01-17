# CPI

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `CpiContext::new()` | Struct-based instruction (e.g., `Transfer { ... }`) |
| `CpiContext::new_with_signer()` | `.invoke_signed(&[signer])` |
| `anchor_spl::token::transfer()` | `pinocchio_token::instructions::Transfer` |
| `system_program::transfer()` | `pinocchio_system::instructions::Transfer` |

## Basic CPI

```rust
use pinocchio_system::instructions::Transfer;

fn transfer_sol(from: &AccountView, to: &AccountView, amount: u64) -> ProgramResult {
    Transfer {
        from,
        to,
        lamports: amount,
    }.invoke()
}
```

**Anchor equivalent:**
```rust
system_program::transfer(
    CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
        },
    ),
    amount,
)?;
```

## CPI with PDA Signer

```rust
use pinocchio::cpi::{Signer, Seed};
use pinocchio_system::instructions::Transfer;

fn withdraw(
    vault: &AccountView,
    recipient: &AccountView,
    vault_state: &Vault,
    amount: u64,
) -> ProgramResult {
    let bump_bytes = [vault_state.bump];
    let seeds: [Seed; 3] = [
        Seed::from(b"vault"),
        Seed::from(vault_state.authority.as_slice()),
        Seed::from(&bump_bytes),
    ];
    let signer = Signer::from(&seeds);

    Transfer {
        from: vault,
        to: recipient,
        lamports: amount,
    }.invoke_signed(&[signer])
}
```

**Anchor equivalent:**
```rust
let seeds = &[b"vault", authority.as_ref(), &[bump]];
let signer = &[&seeds[..]];

system_program::transfer(
    CpiContext::new_with_signer(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer { from, to },
        signer,
    ),
    amount,
)?;
```

## Token Transfer

```rust
use pinocchio_token::instructions::Transfer;

fn transfer_tokens(
    from: &AccountView,
    to: &AccountView,
    authority: &AccountView,
    amount: u64,
) -> ProgramResult {
    Transfer {
        from,
        to,
        authority,
        amount,
    }.invoke()
}
```

**Anchor equivalent:**
```rust
token::transfer(
    CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        token::Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    ),
    amount,
)?;
```

## Token Transfer with PDA

```rust
use pinocchio_token::instructions::Transfer;
use pinocchio::cpi::{Signer, Seed};

fn vault_transfer(
    vault_token: &AccountView,
    destination: &AccountView,
    vault_pda: &AccountView,
    vault_state: &Vault,
    amount: u64,
) -> ProgramResult {
    let bump_bytes = [vault_state.bump];
    let seeds: [Seed; 3] = [
        Seed::from(b"vault"),
        Seed::from(vault_state.authority.as_slice()),
        Seed::from(&bump_bytes),
    ];
    let signer = Signer::from(&seeds);

    Transfer {
        from: vault_token,
        to: destination,
        authority: vault_pda,
        amount,
    }.invoke_signed(&[signer])
}
```

## Available CPIs

**pinocchio-system:**
- `CreateAccount`, `Transfer`, `Allocate`, `Assign`

**pinocchio-token:**
- `Transfer`, `MintTo`, `Burn`, `InitializeMint2`, `InitializeAccount3`
- `Approve`, `Revoke`, `SetAuthority`, `CloseAccount`

## Sysvars

```rust
use pinocchio::sysvars::{Clock, Rent};

let clock = Clock::get()?;
let timestamp = clock.unix_timestamp;

let rent = Rent::get()?;
let lamports = rent.try_minimum_balance(100)?;
```
