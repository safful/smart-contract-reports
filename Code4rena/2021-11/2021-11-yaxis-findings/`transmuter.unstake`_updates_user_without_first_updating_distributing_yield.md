## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [`Transmuter.unstake` updates user without first updating distributing yield](https://github.com/code-423n4/2021-11-yaxis-findings/issues/66) 

# Handle

cmichel


# Vulnerability details

The `updateAccount` function should capture the latest distributed yield to the Transmuter (stored in `buffer`) and therefore work with the latest `totalDividendPoints` variable.
This variable is updated when running a phase distribution with `runPhasedDistribution`.

Unlike all other function that call `updateAccount`, the `unstake` function does not first run a `runPhasedDistribution` modifer to distribute the latest yield.

## Impact
Users that unstake lose out on some yield by not having their alTokens transmuted.

## Recommended Mitigation Steps
Call `runPhasedDistribution` in `unstake` before the `updateAccount` call, as in `stake` or `transmute`.

