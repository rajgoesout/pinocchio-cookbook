# Instructions

## Anchor → Pinocchio

| Anchor | Pinocchio |
|--------|-----------|
| 8-byte discriminator (hash of name) | 1-byte discriminator (manual) |
| `#[instruction(amount: u64)]` | Manual parsing from `instruction_data` |
| Automatic deserialization | Manual `TryFrom<&[u8]>` |
| IDL-generated client | Manual TypeScript client |

## Discriminator Pattern

Anchor uses 8-byte discriminators derived from instruction names. Pinocchio uses a simple 1-byte index:

```rust
pub fn process_instruction(
    _: &Address,
    accounts: &[AccountView],
    instruction_data: &[u8],
) -> ProgramResult {
    match instruction_data.split_first() {
        Some((&0, rest)) => initialize(accounts, rest),
        Some((&1, rest)) => deposit(accounts, rest),
        Some((&2, rest)) => withdraw(accounts, rest),
        _ => Err(ProgramError::InvalidInstructionData),
    }
}
```

**Anchor equivalent:**
```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Initialize>) -> Result<()> { ... }
    pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> { ... }
    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> { ... }
}
```

## Parsing Instruction Data

Anchor deserializes instruction arguments automatically. In Pinocchio, parse manually:

```rust
fn deposit(accounts: &[AccountView], data: &[u8]) -> ProgramResult {
    // Parse u64 amount
    if data.len() < 8 {
        return Err(ProgramError::InvalidInstructionData);
    }
    let amount = u64::from_le_bytes(data[0..8].try_into().unwrap());

    // Use it
    Ok(())
}
```

## Instruction Data Struct

For complex data, use a struct with `TryFrom`:

```rust
pub struct DepositData {
    pub amount: u64,
    pub memo: u8,
}

impl TryFrom<&[u8]> for DepositData {
    type Error = ProgramError;

    fn try_from(data: &[u8]) -> Result<Self, Self::Error> {
        if data.len() < 9 {
            return Err(ProgramError::InvalidInstructionData);
        }
        Ok(Self {
            amount: u64::from_le_bytes(data[0..8].try_into().unwrap()),
            memo: data[8],
        })
    }
}

// Usage
let data = DepositData::try_from(data)?;
```

## Combined Accounts + Data

Pattern similar to Anchor's `Context<T>` with instruction args:

```rust
pub struct Deposit<'a> {
    pub accounts: DepositAccounts<'a>,
    pub data: DepositData,
}

impl<'a> TryFrom<(&'a [u8], &'a [AccountView])> for Deposit<'a> {
    type Error = ProgramError;

    fn try_from((data, accounts): (&'a [u8], &'a [AccountView])) -> Result<Self, Self::Error> {
        Ok(Self {
            accounts: DepositAccounts::try_from(accounts)?,
            data: DepositData::try_from(data)?,
        })
    }
}

impl Deposit<'_> {
    pub fn process(&self) -> ProgramResult {
        // Business logic here
        Ok(())
    }
}

// In entrypoint
match instruction_data.split_first() {
    Some((&1, rest)) => Deposit::try_from((rest, accounts))?.process(),
    // ...
}
```

## Client Side (TypeScript)

Anchor generates clients from IDL. In Pinocchio, build instructions manually:

```typescript
function createDepositIx(
  program: PublicKey,
  accounts: { vault: PublicKey; user: PublicKey },
  amount: bigint
): TransactionInstruction {
  const data = Buffer.alloc(9);
  data.writeUInt8(1, 0);  // discriminator
  data.writeBigUInt64LE(amount, 1);

  return new TransactionInstruction({
    programId: program,
    keys: [
      { pubkey: accounts.vault, isSigner: false, isWritable: true },
      { pubkey: accounts.user, isSigner: true, isWritable: true },
    ],
    data,
  });
}
```

**Anchor equivalent:**
```typescript
await program.methods.deposit(new BN(amount)).accounts({ vault, user }).rpc();
```
