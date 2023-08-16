## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Functions not returning declared values](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/18) 

# Handle

pauliax


# Vulnerability details

## Impact
function withdrawAll in BaseStrategy declares 'returns (uint256 balance)', however, no actual value is returned. function reinvest in MyStrategy declares to return 'uint256 reinvested', however, it also actually does not return anything so they always get assigned a default value of 0.

## Recommended Mitigation Steps
Either remove the return declarations or return the intended values. Otherwise, it may confuse other protocols that later may want to integrate with you.

