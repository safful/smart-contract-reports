## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Useless multiplication by 1](https://github.com/code-423n4/2021-10-tracer-findings/issues/36) 

# Handle

pauliax


# Vulnerability details

## Impact
Uneccesarry multiplication by 1 here:
  require(initialization._fee < 1 * PoolSwapLibrary.WAD_PRECISION, "Fee >= 100%");

## Recommended Mitigation Steps
  require(initialization._fee < PoolSwapLibrary.WAD_PRECISION, "Fee >= 100%");

