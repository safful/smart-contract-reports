## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [notionalCallback returns no value](https://github.com/code-423n4/2021-08-notional-findings/issues/46) 

# Handle

pauliax


# Vulnerability details

## Impact
function notionalCallback (in NotionalV1ToNotionalV2 and CompoundToNotionalV2) declares to return uint, however, no actual value is returned. 

## Recommended Mitigation Steps
Either remove the return declaration or return the intended value (I assume it may return a value that it gets from depositUnderlyingToken/depositAssetToken). Otherwise, it may confuse other protocols that later may want to integrate with you.

