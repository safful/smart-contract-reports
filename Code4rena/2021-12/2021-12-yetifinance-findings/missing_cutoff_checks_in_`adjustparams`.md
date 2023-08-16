## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing cutoff checks in `adjustParams`](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/199) 

# Handle

cmichel


# Vulnerability details

The `ThreePieceWiseLinearPriceCurve.adjustParams` function does not check that `_cutoff1 <= _cutoff2` and also does not revert in this case.
However, this always indicates an error in how this function should be used.

## Recommended Mitigation Steps
Add a `_cutoff1 <= _cutoff2` check.

