---
tags: solana
---
# Resources
https://www.helius.dev/blog/a-hitchhikers-guide-to-solana-program-security
https://rareskills.io/post/init-if-needed-anchor

# #logic_bugs
## Closing Accounts
Anchor has #[account(close=destination)] constraint, automating the secure closure of accounts by transferring lamports, zeroing data, and setting the closed account discriminator, all in one operation

# #data_validation_flaws
## Duplicate Mutable Accounts
## Missing Ownership Check
## Missing Signer Check
## Remaining Accounts
## Type Cosplay

# #rust_specific_issues
## Unsafe Rust

# #access_control_vulnerabilities

## Account Data Matching
## Lack of Authority Transfer

# #arithmetic_and_precision_errors
## Loss of Precision
## Multiplication After Division
## Saturating Arithmetic Functions
## Rounding Errors
## Overflow and Underflow
# #cross_program_invocation_issues
## Account Reloading
After making a CPI to update an account with new state, the state of the "updated" accounts is not automatically updated post-CPI. Performing actions on the stale data will lead to logical errors or incorrect calculations. Explicitly call Anchor's reload method to reload a given account from storage
## Arbitrary CPI
No validation performed on program address passed in
# #program_derived_addresses_misuse
## Bump Seed Canonicalization
## PDA Sharing

# #frontrunning
## Frontrunning with Jito
## Insecure Initialisation

# #others
## Seed Collision