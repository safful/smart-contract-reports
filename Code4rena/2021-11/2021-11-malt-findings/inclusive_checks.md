## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Inclusive checks](https://github.com/code-423n4/2021-11-malt-findings/issues/356) 

# Handle

pauliax


# Vulnerability details

## Impact
These checks should be inclusive:
```solidity     
  require(amountOut > minOut, "EarlyExit: Insufficient output");
  require(_bps > 0 && _bps < 1000, "Must be between 0-100%");
  require(newThreshold > 0 && newThreshold < 10000, "Threshold must be between 0-100%");
  require(_distance > 0 && _distance < 1000, "Override must be between 0-100%");
```

## Recommended Mitigation Steps
Replace > with >= and < with <= where necesseary.

