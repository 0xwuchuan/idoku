---
tags: bitcoin
---
https://learnmeabitcoin.com/technical/script/

# Output
An output is a package of bitcoins created in a bitcoin transaction
You can create multiple outputs in a transaction, where each output contains an amount of bitcoin and a lock on it. A future transaction can then spend these outputs (as inputs) by unlocking them, and create new outputs with new locks on them.

A UTXO is an unspent transaction output. The balance of an address is the sum of all the UTXOs locked to that address

# Script
Script is a mini programming language used a locking mechanism for outputs in bitcoin transactions
- A locking script (ScriptPubKey) is placed on every transaction output
- An unlocking script (ScriptSig or Witness) must be provided to unlock an output

Every node will combine and run these two scripts for each input in each transaction they receive to make sure they validate.

If the unlocking scripts on inputs do not successfully unlock the locking scripts on the outputs being spent, then the transaction is considered invalid and will not be relayed.