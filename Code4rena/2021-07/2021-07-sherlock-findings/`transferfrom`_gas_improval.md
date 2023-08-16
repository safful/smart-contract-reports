## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- disagree with severity

# [`transferFrom` gas improval](https://github.com/code-423n4/2021-07-sherlock-findings/issues/100) 

# Handle

cmichel


# Vulnerability details

The `SherXERC20.transferFrom` function reads the allowance from memory twice.
It should be read once, cached, and then use that value for the `if(cachedAllowance ...)` and for the `newApproval = ...` expressions.

