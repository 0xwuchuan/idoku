---
tags: evm, lending, 
---
# Background
## Attack Transaction
https://app.blocksec.com/explorer/tx/eth/0xa02b159fb438c8f0fb2a8d90bc70d8b2273d06b55920b26f637cab072b7a0e3e
## Summary
Some older compound cTokens return false on failure instead of reverting. A victim staking contract got exploited because of neglecting this

# Attack Steps
1. Update epoch of cUSDC to latest
2. Attacker calls deposit on Victim Staking contract (1,286,577 cUSDC)
	- Here the transferFrom calls fails, but does not revert and instead returns false
3. Staking contract mistakes the deposit as a successful one
4. Attacker calls emergencyWithdraw to drain all the cUSDC in the Staking Contract




