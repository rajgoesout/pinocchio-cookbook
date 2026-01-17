# Token Vault Example

PDA-controlled token custody with deposits and withdrawals.

## Anchor Equivalent

```rust
#[program]
pub mod vault {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.vault.authority = ctx.accounts.payer.key();
        ctx.accounts.vault.mint = ctx.accounts.mint.key();
        ctx.accounts.vault.total = 0;
        ctx.accounts.vault.bump = ctx.bumps.vault;
        Ok(())
    }

    pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
        token::transfer(ctx.accounts.transfer_ctx(), amount)?;
        ctx.accounts.vault.total += amount;
        Ok(())
    }

    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
        token::transfer(ctx.accounts.transfer_ctx_with_signer(), amount)?;
        ctx.accounts.vault.total -= amount;
        Ok(())
    }
}
```

## State

```rust
#[repr(C)]
pub struct Vault {
    pub authority: [u8; 32],
    pub mint: [u8; 32],
    pub total: u64,
    pub bump: u8,
}

impl Vault {
    pub const LEN: usize = 73;

    pub fn from_account_mut(account: &AccountView) -> Result<&mut Self, ProgramError> {
        let mut data = account.try_borrow_mut()?;
        Ok(unsafe { &mut *(data.as_mut_ptr() as *mut Self) })
    }
}
```

## Initialize

```rust
fn initialize(program_id: &Address, accounts: &[AccountView]) -> ProgramResult {
    let [payer, vault, vault_token, mint, _token_program, _system_program, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    let (vault_pda, vault_bump) = Address::find_program_address(
        &[b"vault", mint.address().as_ref()],
        program_id,
    );

    let vault_bump_bytes = [vault_bump];
    let vault_seeds: [Seed; 3] = [
        Seed::from(b"vault"),
        Seed::from(mint.address().as_ref()),
        Seed::from(&vault_bump_bytes),
    ];

    let rent = Rent::get()?;

    // Create vault PDA
    CreateAccount {
        from: payer,
        to: vault,
        lamports: rent.try_minimum_balance(Vault::LEN)?,
        space: Vault::LEN as u64,
        owner: program_id,
    }.invoke_signed(&[Signer::from(&vault_seeds)])?;

    // Create vault token account
    let (token_pda, token_bump) = Address::find_program_address(
        &[b"vault_token", vault.address().as_ref()],
        program_id,
    );
    let token_bump_bytes = [token_bump];
    let token_seeds: [Seed; 3] = [
        Seed::from(b"vault_token"),
        Seed::from(vault.address().as_ref()),
        Seed::from(&token_bump_bytes),
    ];

    CreateAccount {
        from: payer,
        to: vault_token,
        lamports: rent.try_minimum_balance(165)?,
        space: 165,
        owner: &pinocchio_token::ID,
    }.invoke_signed(&[Signer::from(&token_seeds)])?;

    InitializeAccount3 {
        account: vault_token,
        mint,
        owner: vault.address().as_ref(),
    }.invoke()?;

    // Init state
    let state = Vault::from_account_mut(vault)?;
    state.authority.copy_from_slice(payer.address().as_ref());
    state.mint.copy_from_slice(mint.address().as_ref());
    state.total = 0;
    state.bump = vault_bump;

    Ok(())
}
```

## Deposit

```rust
fn deposit(accounts: &[AccountView], amount: u64) -> ProgramResult {
    let [vault, vault_token, user, user_token, _token_program, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    if !user.is_signer() {
        return Err(ProgramError::MissingRequiredSignature);
    }

    Transfer {
        from: user_token,
        to: vault_token,
        authority: user,
        amount,
    }.invoke()?;

    let state = Vault::from_account_mut(vault)?;
    state.total = state.total.checked_add(amount).ok_or(ProgramError::Arithmetic)?;

    Ok(())
}
```

## Withdraw

```rust
fn withdraw(accounts: &[AccountView], amount: u64) -> ProgramResult {
    let [vault, vault_token, user, user_token, _token_program, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    if !user.is_signer() {
        return Err(ProgramError::MissingRequiredSignature);
    }

    let state = Vault::from_account_mut(vault)?;

    if state.authority != *user.address().as_ref() {
        return Err(ProgramError::InvalidAccountData);
    }

    let bump_bytes = [state.bump];
    let seeds: [Seed; 3] = [
        Seed::from(b"vault"),
        Seed::from(state.mint.as_slice()),
        Seed::from(&bump_bytes),
    ];

    Transfer {
        from: vault_token,
        to: user_token,
        authority: vault,
        amount,
    }.invoke_signed(&[Signer::from(&seeds)])?;

    state.total = state.total.checked_sub(amount).ok_or(ProgramError::Arithmetic)?;

    Ok(())
}
```

## Client (TypeScript)

```typescript
const PROGRAM_ID = new PublicKey('Vault11111111111111111111111111111111111111');

function getVaultPDA(mint: PublicKey) {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('vault'), mint.toBuffer()],
    PROGRAM_ID
  );
}

function getVaultTokenPDA(vault: PublicKey) {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('vault_token'), vault.toBuffer()],
    PROGRAM_ID
  )[0];
}

function depositIx(vault: PublicKey, user: PublicKey, userToken: PublicKey, amount: bigint) {
  const vaultToken = getVaultTokenPDA(vault);
  const data = Buffer.alloc(9);
  data.writeUInt8(1, 0);
  data.writeBigUInt64LE(amount, 1);

  return new TransactionInstruction({
    programId: PROGRAM_ID,
    keys: [
      { pubkey: vault, isSigner: false, isWritable: true },
      { pubkey: vaultToken, isSigner: false, isWritable: true },
      { pubkey: user, isSigner: true, isWritable: false },
      { pubkey: userToken, isSigner: false, isWritable: true },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    data,
  });
}
```
