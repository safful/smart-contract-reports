## Tags

- bug
- 0 (Non-critical)
- mStableYieldSource
- sponsor confirmed
- disagree with severity

# [Sponsored event not used](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/1) 

# Handle

tensors


# Vulnerability details

## Impact
The sponsored event is declared but never used.

## Proof of Concept
https://github.com/pooltogether/pooltogether-mstable/blob/0bcbd363936fadf5830e9c48392415695896ddb5/contracts/yield-source/MStableYieldSource.sol#L27

## Recommended Mitigation Steps
Remove the unused event.

