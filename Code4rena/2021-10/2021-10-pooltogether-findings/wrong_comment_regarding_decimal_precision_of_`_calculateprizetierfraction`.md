## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)
- resolved

# [Wrong comment regarding decimal precision of `_calculatePrizeTierFraction`](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/22) 

# Handle

cmichel


# Vulnerability details

The `_calculatePrizeTierFraction` docs say "returns the fraction of the total prize (base 1e18)", but it's base 1e9.
Code seems to be correct.

