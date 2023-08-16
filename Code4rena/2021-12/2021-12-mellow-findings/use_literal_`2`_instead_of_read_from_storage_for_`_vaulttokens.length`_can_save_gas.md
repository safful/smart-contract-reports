## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Use literal `2` instead of read from storage for `_vaultTokens.length` can save gas](https://github.com/code-423n4/2021-12-mellow-findings/issues/104) 

# Handle

WatchPug


# Vulnerability details

The current design requires the number of `_vaultTokens` to be 2 in `UniV3Vault`, therefore `_vaultTokens.length` can be replaced with literal `2` to save ~800 gas from storage read (`SLOAD` after Berlin).

Instances include:

https://github.com/code-423n4/2021-12-mellow/blob/6679e2dd118b33481ee81ad013ece4ea723327b5/mellow-vaults/contracts/UniV3Vault.sol#L101-L101

```solidity=101
tokenAmounts = new uint256[](_vaultTokens.length);
```

