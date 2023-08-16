## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [claimYeti inclusive check](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/300) 

# Handle

pauliax


# Vulnerability details

## Impact
condition should be inclusive >= :
```solidity
  if (available > totalClaimed.add(_amount))
```

