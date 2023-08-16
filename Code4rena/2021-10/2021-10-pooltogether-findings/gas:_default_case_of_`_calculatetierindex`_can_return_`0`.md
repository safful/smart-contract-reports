## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Gas: Default case of `_calculateTierIndex` can return `0`](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/20) 

# Handle

cmichel


# Vulnerability details

If all masks match the `DrawCalculator._calculateTierIndex` function returns `masksLength - numberOfMatches` but it will always be zero at this point as `masksLength == numberOfMatches`.
So just returning zero here would lead to saving a checked subtraction.

