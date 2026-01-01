# Token Transfer CPI

The Anchor way of making a CPI involves creating `CpiContext`, then calling the token transfer function:

```rust
let cpi_accounts = Transfer {
    from: ctx.accounts.owner.to_account_info(),
    to: ctx.accounts.vault.to_account_info(),
    authority: ctx.accounts.owner.to_account_info(),
};

let cpi_program = ctx.accounts.token_program.to_account_info();
let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);

token::transfer(cpi_ctx, amount)?;
```

Pinocchio is simpler in this case:

```rust
Transfer {
    from: self.accounts.owner,
    to: self.accounts.vault,
    lamports: self.instruction_datas.amount,
}
.invoke()?;
```

Note than even in Anchor, we can use the `system_instruction::transfer` function followed by `invoke` from the `solana_program` crate.
