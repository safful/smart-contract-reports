## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [[Gas] Use at least 0.8.0 instead of 0.8.4](https://github.com/code-423n4/2021-06-tracer-findings/issues/37) 

# Handle

hrkrshnn


# Vulnerability details

## Impact

Gas optimization.

## Use at least 0.8.4 instead of 0.8.0

It has an important optimization improvement: a low level inliner.
Especially since you have several small functions.

The current hardhat config indicates that version `0.8.0` is being used.



