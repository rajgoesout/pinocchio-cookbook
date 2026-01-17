# Tokens

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `anchor_spl::token` | `pinocchio-token` |
| `anchor_spl::associated_token` | `pinocchio-associated-token-account` |
| `token::mint_to(ctx, amount)` | `MintTo { ... }.invoke()` |
| `token::transfer(ctx, amount)` | `Transfer { ... }.invoke()` |
| `Account<'info, Mint>` | `AccountView` + manual validation |
| `Account<'info, TokenAccount>` | `AccountView` + manual validation |

## Dependencies

```toml
[dependencies]
pinocchio-token = "0.4"
pinocchio-associated-token-account = "0.3"
```

## Create Mint

```rust
use pinocchio_system::instructions::CreateAccount;
use pinocchio_token::instructions::InitializeMint2;
use pinocchio::sysvars::Rent;

fn create_mint(
    payer: &AccountView,
    mint: &AccountView,
    authority: &Address,
    decimals: u8,
) -> ProgramResult {
    let rent = Rent::get()?;

    CreateAccount {
        from: payer,
        to: mint,
        lamports: rent.try_minimum_balance(82)?,  // Mint size
        space: 82,
        owner: &pinocchio_token::ID,
    }.invoke()?;

    InitializeMint2 {
        mint,
        decimals,
        mint_authority: authority.as_ref(),
        freeze_authority: None,
    }.invoke()
}
```

## Create Token Account

```rust
use pinocchio_token::instructions::InitializeAccount3;

fn create_token_account(
    payer: &AccountView,
    account: &AccountView,
    mint: &AccountView,
    owner: &Address,
) -> ProgramResult {
    let rent = Rent::get()?;

    CreateAccount {
        from: payer,
        to: account,
        lamports: rent.try_minimum_balance(165)?,  // Token account size
        space: 165,
        owner: &pinocchio_token::ID,
    }.invoke()?;

    InitializeAccount3 {
        account,
        mint,
        owner: owner.as_ref(),
    }.invoke()
}
```

## Create ATA

```rust
use pinocchio_associated_token_account::instructions::Create;

fn create_ata(
    payer: &AccountView,
    ata: &AccountView,
    owner: &AccountView,
    mint: &AccountView,
    system_program: &AccountView,
    token_program: &AccountView,
) -> ProgramResult {
    Create {
        funding_account: payer,
        account: ata,
        wallet: owner,
        mint,
        system_program,
        token_program,
    }.invoke()
}
```

**Anchor equivalent:**
```rust
#[derive(Accounts)]
pub struct CreateAta<'info> {
    #[account(
        init,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = owner
    )]
    pub ata: Account<'info, TokenAccount>,
}
```

## Transfer Tokens

```rust
use pinocchio_token::instructions::Transfer;

Transfer {
    from: source,
    to: destination,
    authority: owner,
    amount: 1_000_000,  // 1 token with 6 decimals
}.invoke()?;
```

## Mint Tokens

```rust
use pinocchio_token::instructions::MintTo;

MintTo {
    mint,
    account: destination,
    mint_authority: authority,
    amount: 1_000_000,
}.invoke()?;
```

## Burn Tokens

```rust
use pinocchio_token::instructions::Burn;

Burn {
    account: token_account,
    mint,
    authority: owner,
    amount: 500_000,
}.invoke()?;
```

## Close Token Account

```rust
use pinocchio_token::instructions::CloseAccount;

CloseAccount {
    account: token_account,
    destination: recipient,  // Receives remaining lamports
    authority: owner,
}.invoke()?;
```

## Read Token Account State

```rust
use pinocchio_token::state::TokenAccount;

fn get_balance(account: &AccountView) -> Result<u64, ProgramError> {
    let data = account.data();
    if data.len() < 165 {
        return Err(ProgramError::InvalidAccountData);
    }
    let token = unsafe { &*(data.as_ptr() as *const TokenAccount) };
    Ok(token.amount())
}
```

## Validate Token Account

```rust
fn require_token_account(account: &AccountView) -> ProgramResult {
    if !account.owned_by(&pinocchio_token::ID) || account.data_len() != 165 {
        return Err(ProgramError::InvalidAccountOwner);
    }
    Ok(())
}

fn require_mint(account: &AccountView) -> ProgramResult {
    if !account.owned_by(&pinocchio_token::ID) || account.data_len() != 82 {
        return Err(ProgramError::InvalidAccountOwner);
    }
    Ok(())
}
```

**Anchor equivalent:**
```rust
// Anchor validates automatically with typed accounts
pub token_account: Account<'info, TokenAccount>,
pub mint: Account<'info, Mint>,
```
