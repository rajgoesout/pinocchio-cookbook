# Errors

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| `#[error_code]` derive | Manual enum + `From<MyError>` impl |
| `err!(MyError::X)` | `return Err(MyError::X.into())` |
| `require!(cond, MyError::X)` | Manual `require!` macro or inline `if` |
| Error code starting at 6000 | Same convention (6000+) |

## Custom Error Enum

```rust
use pinocchio::error::ProgramError;

#[repr(u32)]
pub enum MyError {
    AlreadyInitialized = 6000,
    NotInitialized = 6001,
    InvalidAuthority = 6002,
    InsufficientFunds = 6003,
}

impl From<MyError> for ProgramError {
    fn from(e: MyError) -> Self {
        ProgramError::Custom(e as u32)
    }
}

// Usage
return Err(MyError::InvalidAuthority.into());
```

**Anchor equivalent:**
```rust
#[error_code]
pub enum MyError {
    #[msg("Already initialized")]
    AlreadyInitialized,
    #[msg("Not initialized")]
    NotInitialized,
}

// Usage
err!(MyError::AlreadyInitialized)
```

## With thiserror

```toml
[dependencies]
thiserror = { version = "2.0", default-features = false }
```

```rust
use thiserror::Error;

#[derive(Clone, Debug, Error)]
#[repr(u32)]
pub enum VaultError {
    #[error("Already initialized")]
    AlreadyInitialized = 6000,

    #[error("Invalid authority")]
    InvalidAuthority = 6001,

    #[error("Insufficient funds")]
    InsufficientFunds = 6002,
}

impl From<VaultError> for ProgramError {
    fn from(e: VaultError) -> Self {
        ProgramError::Custom(e as u32)
    }
}
```

## Common Errors

```rust
use pinocchio::error::ProgramError;

// Built-in errors
ProgramError::NotEnoughAccountKeys      // Missing accounts
ProgramError::InvalidAccountData        // Bad data
ProgramError::InvalidAccountOwner       // Wrong owner
ProgramError::MissingRequiredSignature  // Not signed
ProgramError::InvalidInstructionData    // Bad instruction
ProgramError::Arithmetic                // Overflow
```

## Require Macro

Anchor has a built-in `require!` macro. In Pinocchio, define your own:

```rust
macro_rules! require {
    ($cond:expr, $err:expr) => {
        if !$cond { return Err($err.into()); }
    };
}

// Usage
require!(account.is_signer(), ProgramError::MissingRequiredSignature);
require!(vault.amount >= withdrawal, MyError::InsufficientFunds);
```

**Anchor equivalent:**
```rust
require!(account.is_signer, MyError::Unauthorized);
require!(vault.amount >= withdrawal, MyError::InsufficientFunds);
```
