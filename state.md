# State

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `#[account]` macro | `#[repr(C)]` struct + manual impl |
| `#[derive(InitSpace)]` | Manual `const LEN` calculation |
| Borsh serialization | Zero-copy pointer cast |
| `Account<'info, T>` | `unsafe` pointer cast to `&T` |
| 8-byte discriminator | 1-byte discriminator (optional) |

## Zero-Copy Struct

Anchor uses Borsh serialization. Pinocchio casts raw bytes directly to structs:

```rust
use pinocchio::account::{Ref, RefMut};

#[repr(C)]
pub struct Counter {
    pub authority: [u8; 32],
    pub value: u64,
    pub bump: u8,
}

impl Counter {
    pub const LEN: usize = 32 + 8 + 1;  // 41 bytes

    pub fn from_account(account: &AccountView) -> Result<Ref<Self>, ProgramError> {
        if account.data_len() < Self::LEN {
            return Err(ProgramError::InvalidAccountData);
        }
        Ok(Ref::map(account.try_borrow()?, |data| unsafe {
            &*(data.as_ptr() as *const Self)
        }))
    }

    pub fn from_account_mut(account: &AccountView) -> Result<RefMut<Self>, ProgramError> {
        let data = account.try_borrow_mut()?;
        if data.len() < Self::LEN {
            return Err(ProgramError::InvalidAccountData);
        }
        Ok(RefMut::map(data, |data| unsafe {
            &mut *(data.as_mut_ptr() as *mut Self)
        }))
    }

    pub fn authority(&self) -> Address {
        Address::from(self.authority)
    }
}
```

**Anchor equivalent:**
```rust
#[account]
pub struct Counter {
    pub authority: Pubkey,
    pub value: u64,
    pub bump: u8,
}
```

## Reading State

```rust
fn read(account: &AccountView) -> Result<u64, ProgramError> {
    let counter = Counter::from_account(account)?;
    Ok(counter.value)
}
```

## Writing State

```rust
fn initialize(account: &AccountView, authority: &Address, bump: u8) -> ProgramResult {
    let counter = Counter::from_account_mut(account)?;
    counter.authority.copy_from_slice(authority.as_ref());
    counter.value = 0;
    counter.bump = bump;
    Ok(())
}

fn increment(account: &AccountView) -> ProgramResult {
    let counter = Counter::from_account_mut(account)?;
    counter
        .value
        .checked_add(1)
        .ok_or(ProgramError::ArithmeticOverflow)?;
    Ok(())
}
```

## Field Ordering (Minimize Padding)

Order fields from largest to smallest to minimize padding:

```rust
// GOOD: Largest to smallest
#[repr(C)]
struct Good {
    big: u64,      // 8 bytes
    medium: u16,   // 2 bytes
    small: u8,     // 1 byte
}  // = 16 bytes (5 padding)

// BAD: Wastes space
#[repr(C)]
struct Bad {
    small: u8,     // 1 + 7 padding
    big: u64,      // 8 bytes
    medium: u16,   // 2 + 6 padding
}  // = 24 bytes (13 padding)
```

## Zero-Padding (Maximum Efficiency)

Store integers as byte arrays to eliminate all padding:

```rust
#[repr(C)]
struct Compact {
    amount: [u8; 8],   // Store u64 as bytes
    fee: [u8; 2],      // Store u16 as bytes
    flag: u8,
}  // = 11 bytes exactly

impl Compact {
    pub fn amount(&self) -> u64 { u64::from_le_bytes(self.amount) }
    pub fn set_amount(&mut self, v: u64) { self.amount = v.to_le_bytes(); }
    pub fn fee(&self) -> u16 { u16::from_le_bytes(self.fee) }
    pub fn set_fee(&mut self, v: u16) { self.fee = v.to_le_bytes(); }
}
```

## Discriminator for Account Types

Anchor auto-generates 8-byte discriminators. In Pinocchio, use a 1-byte type tag:

```rust
#[repr(C)]
pub struct Vault {
    pub discriminator: u8,  // 1 = Vault, 2 = UserRecord, etc.
    pub authority: [u8; 32],
    // ...
}

impl Vault {
    pub const DISCRIMINATOR: u8 = 1;

    pub fn from_account(account: &AccountView) -> Result<Ref<Self>, ProgramError> {
        let data = account.try_borrow()?;
        if data.len() < Self::LEN || data[0] != Self::DISCRIMINATOR {
            return Err(ProgramError::InvalidAccountData);
        }
        Ok(Ref::map(data, |data| unsafe { &*(data.as_ptr() as *const Self) }))
    }
}
```

## Creating Accounts

Anchor's `init` constraint handles account creation. In Pinocchio, use `CreateAccount` CPI:

```rust
use pinocchio_system::instructions::CreateAccount;
use pinocchio::sysvars::Rent;

fn create(payer: &AccountView, new_account: &AccountView, program_id: &Address) -> ProgramResult {
    let rent = Rent::get()?;
    let lamports = rent.try_minimum_balance(Counter::LEN)?;

    CreateAccount {
        from: payer,
        to: new_account,
        lamports,
        space: Counter::LEN as u64,
        owner: program_id,
    }.invoke()
}
```

**Anchor equivalent:**
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + Counter::INIT_SPACE
    )]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```
