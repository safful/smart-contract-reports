## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [The `averagePriceForPeriod` function may revert without proper error message returned](https://github.com/code-423n4/2021-06-tracer-findings/issues/140) 

# Handle

shw


# Vulnerability details

## Impact

The `averagePriceForPeriod` function of `LibPrices` does not handle the case where `j` equals 0 (i.e., no trades happened in the last 24 hours). The transaction reverts due to dividing by 0 without a proper error message returned.

## Proof of Concept

Referenced code:
[LibPrices.sol#L73](https://github.com/code-423n4/2021-06-tracer/blob/main/src/contracts/lib/LibPrices.sol#L73)

## Recommended Mitigation Steps

Add `require(j > 0, "...")` before line 73 to handle this special case.

