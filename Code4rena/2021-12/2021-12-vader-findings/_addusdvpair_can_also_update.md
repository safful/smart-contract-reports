## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- LiquidityBasedTWAP

# [_addUSDVPair can also update](https://github.com/code-423n4/2021-12-vader-findings/issues/185) 

# Handle

pauliax


# Vulnerability details

## Impact
function _addUSDVPair does not check if the foreignAsset does not exist yet, thus it is possible to override it. 

## Recommended Mitigation Steps
Make sure this is the intended behavior or else add validations, e.g.
```solidity
  require(pairData.updatePeriod == 0, "...");
```

