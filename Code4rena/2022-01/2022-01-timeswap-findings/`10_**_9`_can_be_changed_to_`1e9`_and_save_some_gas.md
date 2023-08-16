## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [`10 ** 9` can be changed to `1e9` and save some gas](https://github.com/code-423n4/2022-01-timeswap-findings/issues/177) 

# Handle

WatchPug


# Vulnerability details

https://github.com/code-423n4/2022-01-timeswap/blob/bf50d2a8bb93a5571f35f96bd74af54d9c92a210/Timeswap/Timeswap-V1-Convenience/contracts/libraries/NFTTokenURIScaffold.sol#L132-L132

```solidity
if (significantDigits > 10 ** 9) {
```

Can be changed to:

```solidity
if (significantDigits > 1e9) {
```

