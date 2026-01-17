# Accounts

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `AccountInfo<'info>` | `AccountView` |
| `#[account(mut)]` | Manual `is_writable()` check |
| `#[account(signer)]` | Manual `is_signer()` check |
| `#[account(owner = ...)]` | Manual `owned_by()` check |
| `#[derive(Accounts)]` | `TryFrom<&[AccountView]>` |

## AccountView API

```rust
use pinocchio::{AccountView, Address};

fn process_instruction(accounts: &[AccountView]) {
    let account = &accounts[0];

    // Read fields
    let addr: &Address = account.address();
    let lamports: u64 = account.lamports();
    let data: &[u8] = account.data();
    let owner: &Address = account.owner();

    // Checks
    let is_signer: bool = account.is_signer();
    let is_writable: bool = account.is_writable();
    account.owned_by(&some_program_id);  // bool
}
```

## Destructuring Accounts

```rust
fn process_instruction(accounts: &[AccountView]) -> ProgramResult {
    let [payer, counter, authority, system_program, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };
    Ok(())
}
```

**Anchor equivalent:**
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(mut)]
    pub counter: Account<'info, Counter>,
    pub authority: AccountInfo<'info>,
    pub system_program: Program<'info, System>,
}
```

## Common Validations

```rust
// Signer check (#[account(signer)] in Anchor)
if !authority.is_signer() {
    return Err(ProgramError::MissingRequiredSignature);
}

// Owner check (#[account(owner = program)] in Anchor)
if !account.owned_by(program_id) {
    return Err(ProgramError::InvalidAccountOwner);
}

// Address check (#[account(address = EXPECTED)] in Anchor)
if account.address() != &expected_address {
    return Err(ProgramError::InvalidAccountData);
}

// System account (wallet)
if !account.owned_by(&pinocchio_system::ID) {
    return Err(ProgramError::InvalidAccountOwner);
}

// Token account
if !account.owned_by(&pinocchio_token::ID) {
    return Err(ProgramError::InvalidAccountOwner);
}
```

## Mutable Data Access

```rust
fn update(account: &AccountView) -> ProgramResult {
    let mut data = account.try_borrow_mut()?;
    data[0..8].copy_from_slice(&42u64.to_le_bytes());
    Ok(())
}
```

## TryFrom Pattern

Anchor's `#[derive(Accounts)]` generates validation logic. In Pinocchio, use `TryFrom`:

```rust
pub struct DepositAccounts<'a> {
    pub user: &'a AccountView,
    pub vault: &'a AccountView,
}

impl<'a> TryFrom<&'a [AccountView]> for DepositAccounts<'a> {
    type Error = ProgramError;

    fn try_from(accounts: &'a [AccountView]) -> Result<Self, Self::Error> {
        let [user, vault, _system, ..] = accounts else {
            return Err(ProgramError::NotEnoughAccountKeys);
        };

        if !user.is_signer() {
            return Err(ProgramError::MissingRequiredSignature);
        }

        Ok(Self { user, vault })
    }
}

// Usage
let accounts = DepositAccounts::try_from(accounts)?;
```

**Anchor equivalent:**
```rust
#[derive(Accounts)]
pub struct Deposit<'info> {
    #[account(mut, signer)]
    pub user: AccountInfo<'info>,
    #[account(mut)]
    pub vault: Account<'info, Vault>,
}
```
