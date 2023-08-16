## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Cache decimals](https://github.com/code-423n4/2021-11-malt-findings/issues/371) 

# Handle

pauliax


# Vulnerability details

## Impact
Consider caching decimals when initializing malt and collateralToken to avoid repeated external calls, as they are not supposed to change unless initialized again:
```solidity
  uint256 maltDecimals = malt.decimals();
  uint256 decimals = collateralToken.decimals();
```


