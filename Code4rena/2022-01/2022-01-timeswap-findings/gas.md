## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [gas](https://github.com/code-423n4/2022-01-timeswap-findings/issues/36) 

# Handle

danb


# Vulnerability details

https://github.com/code-423n4/2022-01-timeswap/blob/main/Timeswap/Timeswap-V1-Core/contracts/libraries/MintMath.sol#L65

```
y <= x
```
can be removed

