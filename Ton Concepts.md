---
tags: ton
---
### Bounceable vs Non-Bounceable Addresses
- There can be different representations of an address (raw)
- If destination smart contract does not exist, or if an issue happens during transaction, the message will be "bounced" back to the sender and constitute the remainder of the original value of the transaction
- `bounceable = false` generally means receiver is a wallet
- `bounceable = true` denotes a custom smart contract with its own application logic

https://docs.ton.org/learn/overviews/addresses#bounceable-vs-non-bounceable-addresses

- When a message gets bounced, `bounce` is set to `false` and `bounced` is set to `true`
- The contract should check the `bounced` flag of all inbound messages and either silently accept them or perform some special processing to detect which outbound query has failed. The query contained in the body of a bounced message should never be executed
    
### Getters
```Plain
get fun getter_name: return_type
```

If get is omitted, external users will not be able to call this function and essentially becomes a private method of the contract

- A contract cannot execute a getter of another contract
    

The only way for contracts to communicate on-chain is by sending messages to each other. Messages are handled in _receivers_.

> **Info**: TON Blockchain is an asynchronous blockchain, which means that smart contracts can interact with each other only by sending messages.

### Receivers
```Plain
message message_name {
    message_fields
}
```

- Messages above are considered binary messages, not human readable (serialized into binary data)
    

Textual messages provide human readable messages but cannot hold any arguments (coming to Tact)

### Structs
Structs and messages are almost identical with the only difference that messages have a 32-bit header containing their unique numeric id. This allows messages to be used with receivers since the contract can tell different types of messages apart based on this id.

### Errors
Similar to Solidity in the way the transaction is reverted, but all communication is via messages. Therefore, TOn will actually return the original message back to the sender and mark it as `bounced`

### Sending Modes
`SendRemainingValue` will add to the outgoing value any excess left from the incoming message after all gas costs are deducted from it.

`SendRemainingBalance` will ignore the outgoing value and send the entire balance of the contract. Note that this will not leave any balance for storage costs, so the contract may be deleted

### Control flow
Tact supports if statements but not switch case

Tact does not support `for` loops, instead use `repeat(int)`, `do {} until ()`, `while()`

### Optionals
Values that may be null can be denoted with `?` behind. You cannot access Optionals without explicitly checking for null values, once null values are checked -> use `!!` to tell the compiler that the value cannot be null

### Arrays and Maps
Arrays are implemented using `map<T, U>`

### Traits
```Plain
contract CONTRACT_NAME with TRAIT_NAME
```

Traits are similar to simplified base classes that potentially add state variables, receivers, getters or contract methods.

https://github.com/tact-lang/tact/tree/main/stdlib/libs

- Deployable trait implements receive({Deploy, queryId})
- Ownable trait: need to add `owner: Address` state variable
    - Define a state variable named `owner: Address` and call `self.requireOwner()` in privileged receivers.
- Ownable - Transferable trait: `ChangeOwner{newOwner: Address}` message which allows the owner to transfer ownership
- Stopppable trait: Implicitly implements the Ownable trait
- Trait may add state variables but should not specify their size (declared in program)
    
### Contracts
- Contract addresses on TON are [derived](https://docs.ton.org/learn/overviews/addresses#account-id) from the initial code of the contract (the compiled bytecode) and the initial data of the contract (the arguments of init)
    
# TVM
https://docs.ton.org/v3/documentation/tvm/tvm-overview

# Jetton (TEP-74)
Jettons are tokens on the TON blockchain - similar to ERC-20 tokens on the Ethereum blockchain

Jettons are implemented on TON using a set of smart contracts (unlike just one on Ethereum)

1. Jetton Master smart contract
    - Stores general information about the jetton (token supply, metadata, etc.)
2. Jetton Wallet smart contract
    -  Stores wallet balance information for specific users
        

Jettons are implemented using two smart contracts because of the underlying architecture of TON (asynchronous messages).

Each user has their own associated Jetton wallet smart contract to prevent the bloat of storage in terms of wallet balance information on a single token smart contract (unlike ethereum's balanceOf mapping)

### Transfer flow

A typical Jetton transfer flow message sequence is as follows (-> denotes messages)
![[Pasted image 20250408104728.png]]
1. Payer wallet contract - (transfer) -> Payer Jetton Wallet

Transfer message contains the following data:

|                        |            |                                                                                                                                                                                                                           |
| ---------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name                   | Type       | Description                                                                                                                                                                                                               |
| `query_id`             | uint64     | Allows applications to link three messaging types: `Transfer`, `Transfer notification` and `Excesses` to each other. For this process to be carried out correctly, it is recommended to **always use a unique query id**. |
| `amount`               | coins      | Total `ton coin` amount, that will be sent with message.                                                                                                                                                                  |
| `destination`          | address    | Address of the new owner of the jettons                                                                                                                                                                                   |
| `response_destination` | address    | Wallet address used to return the remaining ton coins with excesses message.                                                                                                                                              |
| `custom_payload`       | maybe cell | Size always is >= 1 bit. Custom data (which is used by either sender or receiver jetton wallet for inner logic).                                                                                                          |
| `forward_ton_amount`   | coins      | Must be > 0 if you want to send `transfer notification message` with `forward payload`. It's a **part of** **`amount`** **value** and **must be lesser than** **`amount`**                                                |
| `forward_payload`      | maybe cell | Size always is >= 1 bit. If first 32 bits = 0x0 it's just a simple message.                                                                                                                                               |

2. Payee Jetton Wallet - (transfer notification) - > Payee wallet contract.
Only sent if `forward_ton_amount` not zero. Contains the following data

|                   |         |
| ----------------- | ------- |
| Name              | Type    |
| `query_id`        | uint64  |
| `amount`          | coins   |
| `sender`          | address |
| `forward_payload` | cell    |

3. Payee Jetton Wallet - (excesses) -> `response_destination`
Only sent if any TON coins are left after paying the fees. Contains the following data

|            |        |
| ---------- | ------ |
| Name       | Type   |
| `query_id` | uint64 |

https://docs.ton.org/develop/dapps/asset-processing/jettons

# pTON (TEP-161)

https://github.com/ston-fi/pton-contracts?tab=readme-ov-file

https://github.com/ton-blockchain/TEPs/pull/161/files?short_path=dc97be0#diff-dc97be096891c21597eab73dbf308c6acad8d1b7d51021eba715e084932d81a4

# Other Notes

- init() -> constructor()
    
- receive("message") -> similar to fallback but have to declare message and can be multiple receive()
    
- ton() -> multiply by power of 9
    
- TON math is safe by default
    
- Coins -> 120 bit (range 0 to 2^120 - 1 (takes 120 bit = 15 bytes)
    
- If sending textual message, send `string.asComment()`
    
- Timestamps : Int as uint32
    
- The underlying principle of TON smart contracts is that we don't need to have smart contracts deployed yet, we can always just deploy them automatically when sending messsages
    

  

## More reading

https://docs.ton.org/v3/guidelines/smart-contracts/security/secure-programming