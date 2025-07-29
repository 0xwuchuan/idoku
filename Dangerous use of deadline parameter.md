---
tags: evm, uniswap, best_practice
---

https://github.com/code-423n4/2023-12-particle-findings/issues/59

Using block.timestamp as a deadline parameter is effectively a non-operation that has no effect nor protection. Failure to provide a proper deadline value enables pending transactions to be maliciously executed at a later point. Transactions that provide an insufficient amount of gas such that they are not mined within a reasonable amount of time, can be picked by malicious actors or MEV bots and executed later in detriment of the submitter.
