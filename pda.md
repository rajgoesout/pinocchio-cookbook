# PDAs

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `#[account(seeds = [...], bump)]` | `Address::find_program_address()` |
| Automatic PDA derivation | Manual derivation and validation |
| `ctx.bumps.account_name` | Store bump in state or re-derive |
| `seeds::program = other_program` | Pass program_id explicitly |

## Derive PDA

```rust
use pinocchio::Address;

let (pda, bump) = Address::find_program_address(
    &[b"vault", authority.as_ref()],
    program_id,
);
```

**Anchor equivalent:**
```rust
#[account(
    seeds = [b"vault", authority.key().as_ref()],
    bump
)]
pub vault: Account<'info, Vault>,
```

## Validate PDA

```rust
fn validate_pda(
    account: &AccountView,
    authority: &AccountView,
    program_id: &Address,
) -> Result<u8, ProgramError> {
    let (expected, bump) = Address::find_program_address(
        &[b"vault", authority.address().as_ref()],
        program_id,
    );

    if account.address() != &expected {
        return Err(ProgramError::InvalidAccountData);
    }

    Ok(bump)
}
```

## Create PDA Account

Anchor handles PDA creation with `init`. In Pinocchio, derive seeds and call `CreateAccount`:

```rust
use pinocchio::cpi::{Signer, Seed};
use pinocchio_system::instructions::CreateAccount;
use pinocchio::sysvars::Rent;

fn create_pda(
    payer: &AccountView,
    pda_account: &AccountView,
    authority: &AccountView,
    program_id: &Address,
) -> ProgramResult {
    let (expected, bump) = Address::find_program_address(
        &[b"vault", authority.address().as_ref()],
        program_id,
    );

    if pda_account.address() != &expected {
        return Err(ProgramError::InvalidAccountData);
    }

    let rent = Rent::get()?;
    let bump_bytes = [bump];
    let seeds: [Seed; 3] = [
        Seed::from(b"vault"),
        Seed::from(authority.address().as_ref()),
        Seed::from(&bump_bytes),
    ];
    let signer = Signer::from(&seeds);

    CreateAccount {
        from: payer,
        to: pda_account,
        lamports: rent.try_minimum_balance(Vault::LEN)?,
        space: Vault::LEN as u64,
        owner: program_id,
    }.invoke_signed(&[signer])
}
```

**Anchor equivalent:**
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + Vault::INIT_SPACE,
        seeds = [b"vault", authority.key().as_ref()],
        bump
    )]
    pub vault: Account<'info, Vault>,
    pub authority: AccountInfo<'info>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

## Store Bump in State

Avoid re-computing the bump by storing it:

```rust
#[repr(C)]
pub struct Vault {
    pub authority: [u8; 32],
    pub bump: u8,  // Store bump to avoid re-derivation
}

// On init
vault.bump = bump;

// Later, use stored bump for signing
let bump_bytes = [vault.bump];
let seeds: [Seed; 3] = [
    Seed::from(b"vault"),
    Seed::from(vault.authority.as_slice()),
    Seed::from(&bump_bytes),
];
```

## Common PDA Patterns

```rust
// User-specific account (like Anchor's seeds = [b"vault", user.key()])
let (user_vault, _) = Address::find_program_address(
    &[b"vault", user.address().as_ref()],
    program_id,
);

// Config/global account (like Anchor's seeds = [b"config"])
let (config, _) = Address::find_program_address(
    &[b"config"],
    program_id,
);

// Multi-key derivation
let (escrow, _) = Address::find_program_address(
    &[b"escrow", maker.as_ref(), taker.as_ref(), mint.as_ref()],
    program_id,
);
```
