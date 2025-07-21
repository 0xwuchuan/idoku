---
tags: evm, oracle
---
# Summary

This type of vulnerability occurs when a protocol attempts to integrate multiple oracle providers (like Chainlink and Pyth) for price feeds, but the available feed pairs from different providers are incompatible or use different base currencies. The core issue stems from **currency denomination mismatches** between oracle feeds that are expected to provide comparable price data.

# Common Characteristics

**Root Cause**: 
Different oracle providers often support different trading pairs for the same asset. For example, one provider might offer ETH/USD while another only provides ETH/BTC, making direct price comparisons impossible without additional conversion steps.

**Technical Impact**:
- **Broken Core Functionality**: Since oracle modules typically serve as foundational infrastructure for DeFi protocols, incompatible feeds can render entire systems non-functional
- **Price Calculation Errors**: Attempting to compare prices in different denominations leads to incorrect valuations
- **Protocol Deadlock**: Systems may fail to initialize or execute critical operations that depend on oracle price data

# Examples
https://github.com/sherlock-audit/2023-12-flatmoney-judging/blob/main//004-M/273.md