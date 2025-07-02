---
tags: aptos
---
# Accounts
Accounts on Aptos must be explicitly created before executing transactions
- Rotating authentication key: Accounts authentication key can be changed to be controlled via a different private key

There are three types of accounts on Aptos:
1. Standard account: similar to a typical EOA on Ethereum (public/private key)
2. Resource account: Autonomous account without a corresponding private key to store resources or publish modules on-chain
3. Object: Complex set of resources stored within a single address representing a single entity

On Aptos, on-chain state is organized into modules and resources. These modules and resources are then stored within the individual accounts. An account may contain an arbitrary number of Move modules and resources
## Resources
Move resources contain data but no code.

Struct definitions in Move Aptos may include abilities such as `key` or `store`
- The `key` ability allows struct instances to be stored in global storage or directly in an account (resource)
- The `store` ability allows struct instances to be stored within resources

All instances and resources are defined within a module that is stored at an address.

For example
```
module 0x1234::coin { // 0x1234 is the address, coin is the module
    struct CoinStore<phantom CoinType> has key { // resource
        coin: Coin<CoinType>,
    }
	// The use of the phantom type allows for there to exist many distinct types of `CoinStore` resources with different `CoinType` parameters.
 
    struct Coin<phantom CoinType> has store { // instance
	    value: u64,
	}
}
```
## Modules
Move modules contain code but not data. 
## Objects
### Creating an Object

# Transaction


# Global Storage
In pseudocode, the global storage looks something like:
```
module 0x42::example {
  struct GlobalStorage {
    resources: Map<(address, ResourceType), ResourceValue>,
    modules: Map<(address, ModuleName), ModuleBytecode>
  }
}
```
Each address can store both resource data values and module code values. Each address can store at most one resource value of a given type and at most one module with a given name
## Storage Operators
Move programs can create, delete, and update resources in global storage using the following five instructions:
1. `move_to<T>(&signer, T)`: 
2. `move_from<T>(address): T`
3. `borrow_global_mut<T>(address): &mut T`
4. `borrow_global<T>(address): &T`
5. `exists<T>(address): bool`

When a function access a resource using `move_from`, `borrow_global`, or `borrow_global_mut`, the function must indicate that it acquires that resource.

```
module 0x42::example {
 
    struct Balance has key { value: u64 }
 
    public fun add_balance(s: &signer, value: u64) {
        move_to(s, Balance { value })
    }
 
    public fun extract_balance(addr: address): u64 acquires Balance {
        let Balance { value } = move_from<Balance>(addr); // acquires needed
        value
    }
 
    public fun extract_and_add(sender: address, receiver: &signer) acquires Balance {
        let value = extract_balance(sender); // acquires needed here
        add_balance(receiver, value)
    }
}
 
module 0x42::other {
    fun extract_balance(addr: address): u64 {
        0x42::example::extract_balance(addr) // no acquires needed
    }
}
```

# Coin & Fungible Asset
[OtterSec's Guide to Aptos Fungible Assets](https://osec.io/blog/2025-02-10-hitchhikers-guide-to-aptos-fungible-assets)
Store -> FA

Fungible Assets use the "Hot Potato" pattern. For more information of Hot Potato pattern can refer to [[Sui Concepts#store]]. This means that Fungible Assets are ephemeral and must be destroyed at the end of the transaction. How do we track the balances then? 

This is where FungibleStore comes in. FungibleStore stores the metadata, balance and whether the store is frozen. So the deposit and withdraw functions will take in a FungibleAsset and increment/decrement the balance.

There are 3 (situationally 4) entrypoints for users to transfer their tokens
1. fungible_asset::transfer (if there is a withdraw dispatch function exist will revert)
2. primary_fungible_store::transfer (uses dispatchable_fungible_store::transfer)
3. dispatchable_fungible_store::transfer
4. coin::transfer (if migrated from coin to fungible asset)