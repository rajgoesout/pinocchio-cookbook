# Counter Example

A simple counter demonstrating PDA-based state management.

## Anchor Equivalent

```rust
#[program]
pub mod counter {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.counter.authority = ctx.accounts.authority.key();
        ctx.accounts.counter.value = 0;
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        ctx.accounts.counter.value += 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    #[account(init, payer = payer, space = 8 + 32 + 8 + 1, seeds = [b"counter", authority.key().as_ref()], bump)]
    pub counter: Account<'info, Counter>,
    pub authority: AccountInfo<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut, seeds = [b"counter", authority.key().as_ref()], bump)]
    pub counter: Account<'info, Counter>,
    pub authority: Signer<'info>,
}
```

## State

```rust
#[repr(C)]
pub struct Counter {
    pub authority: [u8; 32],
    pub value: u64,
    pub bump: u8,
}

impl Counter {
    pub const LEN: usize = 41;

    pub fn from_account_mut(account: &AccountView) -> Result<&mut Self, ProgramError> {
        let mut data = account.try_borrow_mut()?;
        Ok(unsafe { &mut *(data.as_mut_ptr() as *mut Self) })
    }
}
```

## Program

```rust
#![no_std]

use pinocchio::{AccountView, Address, ProgramResult, entrypoint, error::ProgramError};
use pinocchio::cpi::{Signer, Seed};
use pinocchio::sysvars::Rent;
use pinocchio_system::instructions::CreateAccount;

pub const ID: Address = pinocchio::address!("Counter1111111111111111111111111111111111");

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Address,
    accounts: &[AccountView],
    instruction_data: &[u8],
) -> ProgramResult {
    match instruction_data.first() {
        Some(&0) => initialize(program_id, accounts),
        Some(&1) => increment(accounts),
        _ => Err(ProgramError::InvalidInstructionData),
    }
}

fn initialize(program_id: &Address, accounts: &[AccountView]) -> ProgramResult {
    let [payer, counter, authority, _system, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    let (pda, bump) = Address::find_program_address(
        &[b"counter", authority.address().as_ref()],
        program_id,
    );

    if counter.address() != &pda {
        return Err(ProgramError::InvalidAccountData);
    }

    let bump_bytes = [bump];
    let seeds: [Seed; 3] = [
        Seed::from(b"counter"),
        Seed::from(authority.address().as_ref()),
        Seed::from(&bump_bytes),
    ];

    let rent = Rent::get()?;
    CreateAccount {
        from: payer,
        to: counter,
        lamports: rent.try_minimum_balance(Counter::LEN)?,
        space: Counter::LEN as u64,
        owner: program_id,
    }.invoke_signed(&[Signer::from(&seeds)])?;

    let state = Counter::from_account_mut(counter)?;
    state.authority.copy_from_slice(authority.address().as_ref());
    state.value = 0;
    state.bump = bump;

    Ok(())
}

fn increment(accounts: &[AccountView]) -> ProgramResult {
    let [counter, authority, ..] = accounts else {
        return Err(ProgramError::NotEnoughAccountKeys);
    };

    if !authority.is_signer() {
        return Err(ProgramError::MissingRequiredSignature);
    }

    let state = Counter::from_account_mut(counter)?;

    if state.authority != *authority.address().as_ref() {
        return Err(ProgramError::InvalidAccountData);
    }

    state.value = state.value.checked_add(1).ok_or(ProgramError::Arithmetic)?;
    Ok(())
}
```

## Client (TypeScript)

```typescript
const PROGRAM_ID = new PublicKey('Counter1111111111111111111111111111111111');

function getCounterPDA(authority: PublicKey) {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('counter'), authority.toBuffer()],
    PROGRAM_ID
  )[0];
}

function initializeIx(payer: PublicKey, authority: PublicKey) {
  return new TransactionInstruction({
    programId: PROGRAM_ID,
    keys: [
      { pubkey: payer, isSigner: true, isWritable: true },
      { pubkey: getCounterPDA(authority), isSigner: false, isWritable: true },
      { pubkey: authority, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
    ],
    data: Buffer.from([0]),
  });
}

function incrementIx(authority: PublicKey) {
  return new TransactionInstruction({
    programId: PROGRAM_ID,
    keys: [
      { pubkey: getCounterPDA(authority), isSigner: false, isWritable: true },
      { pubkey: authority, isSigner: true, isWritable: false },
    ],
    data: Buffer.from([1]),
  });
}
```
