---
tags: sui
---
# Object Model
An object is a basic unit of storage on the Sui blockchain. A smart contract is an object (called a Sui Move Package), and these smart contracts manipulate objects on the Sui blockchain. 

An object in Sui is a package (set of Move bytecode modules) or object (typed data structure with fields) with additional metadata detailing its id, version, transaction digest, owner field indicating how this object can be accessed (more on that later)
## Object Metadata
Each Sui object has the following main metadata:
- Unique Id: Each object has an unique identifier which is generated upon the object's creation and is immutable. 
- Version: The version is a unsigned integer that monotonically increases with every transaction that modifies it 
> Every object stored on chain is reference by an ID and version. When a transaction modifies an object, it writes the new contents to an on-chain reference with the same ID but a later version. Only one version of the object is available to transactions
- Owner: Every object is associated with an owner who has control over changes to the object

The other fields of an object are detailed [here](https://docs.sui.io/references/sui-api/sui-graphql/reference/types/objects/object)
## Object Ownership
There are four distinct ownership types for objects:
1. Address owned
2. Immutable
3. Shared
4. Wrapped
### Address Owned
An address-owned object is owned by a specific 32-byte address that is either an account address or an object ID. An address-owned object is accessible only to its owner and no others.
> You can't access an object transferred to an immutable object. Read more [here](https://docs.sui.io/concepts/transfers/transfer-to-object)
### Immutable
An immutable object is an object that can't be mutated, transferred or deleted. Immutable objects have no owner, so anyone can use them.
### Shared
A shared object can be read and modified by any account on Sui. The specific rules of interaction is defined by the implementation of the object. Typical usage of shared objects are marketplaces, escrows where multiple accounts need access to the same state.
### Wrapped
A wrapped object is an object that is stored within another object. The object must have the store ability (more on that later).  When an object is wrapped, the object no longer exists independently on-chain. You can no longer pass the wrapped object as an argument in any way in Sui Move calls. The only access point is through the wrapping object

## Abilities
Each type/struct can be defined with an "ability". Abilities are a way to allow certain behaviours for a type/struct
```move
/// This struct has the `copy` and `drop` abilities.
struct VeryAble has copy, drop {
    // field: Type1,
    // field2: Type2,
    // ...
}
```
Move supports 4 abilities.
### copy
value of this type can be copied (basic types have copy but more complex types have to go by a case by case basis)
### drop
value of this type can be automatically destroyed at the end of scope (usually used for resource types)
### key
value of this type is considered an object and can be used in storage functions. The first field of the struct must be named `id` and have the type `UID`
### store
ability of a type's value to be stored (transferable by anyone using transfer::public_transfer) and able to be wrapped in other objects

A struct without abilities cannot be discarded, copied, or stored in storage. We call such a struct a [Hot Potato](https://move-book.com/programmability/hot-potato-pattern.html), which is a powerful pattern in Sui Move.

# Transactions
There are two kinds of transactions on Sui
1. Programmable Transaction Blocks - anyone can submit on the network
2. System transactions - only validators can directly submit (responsible for keeping the network running)
## Transaction Structure
> Every transaction explicitly specifies the objects it operates on (similar to Accounts in a Solana transaction)

Each transaction consists of the following:
- Sender: account that signs the transaction
- Command inputs: arguments for commands
- Commands: operations to be executed
- Gas object: must be owned by sender and must be of type `sui::coin::Coin<SUI>`
- Gas price and budget: budget being similar to gas limit in Ethereum
> Gas object (input) must have a value higher than `gas price * gas budget`

### Commands
The possible commands are:
- `TransferObjects` sends multiple (one or more) objects to a specified address.
- `SplitCoins` splits off multiple (one or more) coins from a single coin. It can be any `sui::coin::Coin<_>` object.
- `MergeCoins` merges multiple (one or more) coins into a single coin. Any `sui::coin::Coin<_>` objects can be merged, as long as they are all of the same type.
- `MakeMoveVec` creates a vector (potentially empty) of Move values. This is used primarily to construct vectors of Move values to be used as arguments to `MoveCall`.
- `MoveCall` invokes either an `entry` or a `public` Move function in a published package.
- `Publish` creates a new package and calls the `init` function of each module in the package.
- `Upgrade` upgrades an existing package. The upgrade is gated by the `sui::package::UpgradeCap` for that package.



