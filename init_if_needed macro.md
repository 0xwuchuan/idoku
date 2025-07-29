---
tags: solana, 
---
https://rareskills.io/post/init-if-needed-anchor

# Summary
- Solana does not have an "initialized" flag, Anchor determines an account to be already initialized if:
	1. Account has a non-zero lamport balance **OR**
	2. Account is owned by the system program
- init_if_needed needs further inspection for security vulnerabilities if:
	1. There is a way to reduce the lamport balance to zero or transfer ownership to the system program (which allows account to the reinitialized)
	2. The program has both an init macro and init_if_needed macro (two different code paths could result in unexpected state)
- Clearing the account data through realloc (or other methods) does not change the owner or reduce lamports to zero, so calling init again will fail
