## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [FSDVesting: Define new constant LINEAR_VEST_AFTER_CLIFF](https://github.com/code-423n4/2021-11-fairside-findings/issues/30) 

# Handle

hickuphh3


# Vulnerability details

## Impact

`DURATION.sub(CLIFF)` is calculated in `calculateVestingClaim()`. Since both are constants, it would be better to define a new constant `LINEAR_VEST_AFTER_CLIFF` that refers to the vest duration after the cliff.

## Recommended Mitigation Steps

`uint256 private constant LINEAR_VEST_AFTER_CLIFF = 18 * ONE_MONTH;`

