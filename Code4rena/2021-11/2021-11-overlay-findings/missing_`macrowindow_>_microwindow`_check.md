## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing `macroWindow > microWindow` check](https://github.com/code-423n4/2021-11-overlay-findings/issues/80) 

# Handle

cmichel


# Vulnerability details

The `OverlayV1UniswapV3Market.constructor` does not verify that the `marcoWindow > microWindow` but the code implicitly uses this assumption when computing the TWAPs.

## Recommended Mitigation Steps
Validate that `macroWindow > microWindow` in the constructor.


