## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`LendingPair.pendingSupplyInterest` is not accurate](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/124) 

# Handle

cmichel


# Vulnerability details

The `LendingPair.pendingSupplyInterest` does not accrue the new interest since the last update.

## Impact
The returned value is not accurate.

## Recommendation
Accrue it first such that `cumulativeInterestRate` updates and `_newInterest` returns the updated value.

