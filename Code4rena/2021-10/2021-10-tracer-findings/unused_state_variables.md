## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unused state variables](https://github.com/code-423n4/2021-10-tracer-findings/issues/31) 

# Handle

pauliax


# Vulnerability details

## Impact
Not used variables. 
In library PoolSwapLibrary:
  bytes16 public constant zero = 0x00000000000000000000000000000000;
  bytes16 private constant NEGATIVE_ZERO = 0x80000000000000000000000000000000;
In contract PoolKeeper:
  uint256 public constant MAX_DECIMALS = 18;

## Recommended Mitigation Steps
Remove if you don't need them to save some gas.

