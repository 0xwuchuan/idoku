---
tags: solidity
---
Some protocols have issues that should be taken note of, because in certain cases they may lead to exploits

# Lending Protocols

## CompoundV2

CompoundV2 has a precision loss/rounding error in `redeemUnderlying`. This can be exploited if a pool is under utilised (empty), or can be used to make small changes to the exchangeRate and cause an account to be liquidatable

[Lending Protocol Attack Analysis(s)](https://okg-block.larksuite.com/docx/JyWAdzgkMo65GFxrTrRuPG5GsVg)

[20240926 OnyxDAO](https://okg-block.larksuite.com/docx/F05QdNDNoomRiBxYxDsuKvnssud)

[20250212 - zkLend 攻击事件分析](https://okg-block.larksuite.com/wiki/G8hpw3R37is1ErkyEaLuR9eQsTc)
# Yield Aggregators

https://mixbytes.io/blog/yield-aggregators-common-pitfalls?

https://medium.com/amber-group/bsc-flash-loan-attack-pancakebunny-3361b6d814fd

1. Manipulated conversion rate between shares and underlying
    
2. Swaps not protected from sandwich attacks
    
3. Attacks from the owner
    
4. Token-specific risks
    
5. Bad balance management
    
6. Poor vault lifecycle design
    
# EIP7702 Accounts

- Constructors will only execute on the delegate contract (not in the context of the account), therefore, we should use initializers instead similar to Proxies
    
- Initializers should have the relevant protections to prevent malicious users from reinitializing the account
    
- Contracts should implement receive() according to business requirements
    
- address(this) will work different based on the execution context
    
# Token Launchpads

- Pool squatting where attacker deploys pool before and gets initial liquidity

four.meme etc.

# Random

- Anything with "rewards" or "yield" is susceptible to manipulation, ensure that rewards cannot be manipulated through self-transfers, reentrancy etc.

# ERC4626 Vaults
1. [Exchange Rate Manipulation in ERC4626 Vaults](https://www.euler.finance/blog/exchange-rate-manipulation-in-erc4626-vaults)
2. 