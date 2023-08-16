## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [index + 1 can be simplified](https://github.com/code-423n4/2021-11-nested-findings/issues/207) 

# Handle

pauliax


# Vulnerability details

## Impact
This can be simplified to reduce gas costs by eliminating math operation:
```solidity
  // before
  require(_accountIndex + 1 <= shareholders.length, "FeeSplitter: INVALID_ACCOUNT_INDEX");
  // after
  require(_accountIndex < shareholders.length, "FeeSplitter: INVALID_ACCOUNT_INDEX");
 ```

