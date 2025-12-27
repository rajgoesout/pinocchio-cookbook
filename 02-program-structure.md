# Program structure

Anchor analogy: `#[program]` modules become explicit dispatch and entrypoint wiring.

What it expands to (conceptually):

1. Generates the Solana entrypoint (process_instruction)
2. Reads instruction_data
3. Reads the 8-byte Anchor discriminator
4. Matches that discriminator to initialize
5. Deserializes Context<Initialize>
6. Calls initialize(ctx)

```rust
#[program]
pub mod my_program {
    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}
```

Anchor programs organize instructions and account validation for you. In Pinocchio, you split those into explicit modules.

```rust
entrypoint!(process_instruction);

fn process_instruction(
    _program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    match instruction_data.split_first() {
        Some((Instruction1::DISCRIMINATOR, data)) => Instruction1::try_from((data, accounts))?.process(),
        Some((Instruction2::DISCRIMINATOR, _ )) => Instruction2::try_from(accounts)?.process(),
        _ => Err(ProgramError::InvalidInstructionData),
    }
}
```

There are three explicit dispatch points:

1. `entrypoint!(process_instruction);`
   This registers your function as the Solana program entrypoint.
2. `instruction_data.split_first()`
   This is where instruction routing begins.
   First byte = instruction discriminator
   Remaining bytes = instruction payload
3. `match` arms on discriminator
   These lines are the actual dispatcher.
   Each arm:
   - Identifies the instruction
   - Decodes accounts + data
   - Calls .process()
