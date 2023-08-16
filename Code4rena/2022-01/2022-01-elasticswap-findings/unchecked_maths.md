## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unchecked maths](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/177) 

# Handle

pauliax


# Vulnerability details

## Impact
Using the unchecked keyword to avoid redundant arithmetic checks and save gas when an underflow/overflow cannot happen, e.g.:
```solidity
    if (rootK > rootKLast) {
        uint256 numerator =
            _totalSupplyOfLiquidityTokens * (rootK - rootKLast);
```
rootK - rootKLast will never underflow here.


