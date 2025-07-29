---
tags: solana
---

# Resources
https://blog.offside.io/p/token-2022-security-best-practices-part-1
- WSOL has both SPL Token and Token-2022 representations, with the SPL Token being the "canonical" one while the Token-2022 is barely used
- WSOL = So11111111111111111111111111111111111111112

Transfer tokens with a transfer fee (extension) requires using transfer_checked or transfer_checked_with_fee

https://osec.io/blog/2025-05-14-king-of-the-sol

# Anchor Related Notes
[[init_if_needed macro]]

When anchor closes an account it does three main operations
1. Transfer all lamports to destination account
2. Transfers accounts to system program
3. realloc(0, false) to clear out the data

``` rust
pub fn close<'info>(info: AccountInfo<'info>, sol_destination: AccountInfo<'info>) -> Result<()> {
    // Transfer tokens from the account to the sol_destination.
    let dest_starting_lamports = sol_destination.lamports();
    **sol_destination.lamports.borrow_mut() =
        dest_starting_lamports.checked_add(info.lamports()).unwrap();
    **info.lamports.borrow_mut() = 0;

    info.assign(&system_program::ID);
    info.realloc(0, false).map_err(Into::into)
}

pub fn is_closed(info: &AccountInfo) -> bool {
    info.owner == &System::id() && info.data_is_empty()
}
```