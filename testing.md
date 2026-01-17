# Testing

## LiteSVM

Fast in-process Solana simulator.

```toml
[dev-dependencies]
litesvm = "0.6"
solana-sdk = "2.2"
```

```rust
use litesvm::LiteSVM;
use solana_sdk::{
    instruction::{AccountMeta, Instruction},
    pubkey::Pubkey,
    signature::{Keypair, Signer},
    transaction::Transaction,
};

#[test]
fn test_counter() {
    let mut svm = LiteSVM::new();
    let payer = Keypair::new();
    let counter = Keypair::new();

    svm.airdrop(&payer.pubkey(), 10_000_000_000).unwrap();

    // Load program
    let program_id = Pubkey::new_unique();
    let program_bytes = include_bytes!("../../target/deploy/my_program.so");
    svm.add_program(program_id, program_bytes);

    // Build instruction
    let ix = Instruction {
        program_id,
        accounts: vec![
            AccountMeta::new(payer.pubkey(), true),
            AccountMeta::new(counter.pubkey(), true),
        ],
        data: vec![0],  // Initialize
    };

    // Execute
    let tx = Transaction::new_signed_with_payer(
        &[ix],
        Some(&payer.pubkey()),
        &[&payer, &counter],
        svm.latest_blockhash(),
    );

    svm.send_transaction(tx).unwrap();

    // Verify
    let account = svm.get_account(&counter.pubkey()).unwrap();
    assert_eq!(account.data[32..40], 0u64.to_le_bytes());
}
```

## Mollusk

Instruction-level testing without transaction overhead.

```toml
[dev-dependencies]
mollusk-svm = "0.1"
solana-sdk = "2.2"
```

```rust
use mollusk_svm::Mollusk;
use solana_sdk::{account::Account, pubkey::Pubkey};

#[test]
fn test_increment() {
    let program_id = Pubkey::new_unique();
    let mollusk = Mollusk::new(&program_id, "target/deploy/my_program");

    let counter = Pubkey::new_unique();
    let counter_data = vec![0u8; 41];  // Counter::LEN

    let result = mollusk.process_instruction(
        &[1],  // Increment discriminator
        &[(counter, Account {
            lamports: 1_000_000,
            data: counter_data,
            owner: program_id,
            ..Account::default()
        })],
    );

    assert!(result.is_ok());
    let account = &result.unwrap().0[0];
    assert_eq!(&account.data[32..40], &1u64.to_le_bytes());
}
```

## Test Organization

```
tests/
├── integration.rs    # Full transaction tests (LiteSVM)
└── unit.rs          # Instruction tests (Mollusk)
```

## Running Tests

```bash
cargo test-sbf           # Build + test
cargo test               # Unit tests only
```
