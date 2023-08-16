## Tags

- bug
- G (Gas Optimization)
- mStableYieldSource
- sponsor confirmed

# [[Optimization] Use 0.8.4 in MStableYieldSource.sol](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/65) 

# Handle

hrkrshnn


# Vulnerability details

## Use 0.8.4

The version 0.8.4 includes an important low level inliner that can save
gas. Upgrading `MStableYieldSource.sol` from 0.8.2 to 0.8.4 should
improve gas.


