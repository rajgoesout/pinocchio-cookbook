# Accounts and constraints

`#[derive(Accounts)]` generates TryFrom logic for you

`#[account(mut, signer)]` becomes runtime checks

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut, signer)]
    pub owner: AccountInfo<'info>,

    #[account(mut)]
    pub vault: AccountInfo<'info>,
}
```

This is the same validation, just written out explicitly.

```rust
impl<'a> TryFrom<&'a [AccountInfo]> for Initialize<'a> {
    type Error = ProgramError;

    fn try_from(accounts: &'a [AccountInfo]) -> Result<Self, Self::Error> {
        let [owner, vault, ..] = accounts else {
            return Err(ProgramError::NotEnoughAccountKeys);
        };

        if !owner.is_signer() {
            return Err(ProgramError::InvalidAccountOwner);
        }

        Ok(Self { owner, vault })
    }
}
```
