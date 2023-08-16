## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Unnecessary length check in `Swap.constructor`](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/217) 

# Handle

cmichel


# Vulnerability details

The `Swap.constructor` checks if both arrays `_pooledTokens` and `decimals` are of length two, but then does another check if these arrays have the same length.

```solidity
require(
    _pooledTokens.length == decimals.length,
    "_pooledTokens decimals mismatch"
);
```

This check will always be true as it has been checked that both arrays are of length two.

