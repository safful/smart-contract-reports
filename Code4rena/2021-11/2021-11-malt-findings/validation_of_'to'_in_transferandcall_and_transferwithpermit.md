## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Validation of 'to' in transferAndCall and transferWithPermit](https://github.com/code-423n4/2021-11-malt-findings/issues/350) 

# Handle

pauliax


# Vulnerability details

## Impact
In functions transferAndCall and transferWithPermit the condition should be AND, not OR:
```solidity
  require(to != address(0) || to != address(this));
```

## Recommended Mitigation Steps
```solidity
  require(to != address(0) && to != address(this));
```


