## Tags

- bug
- G (Gas Optimization)
- sponsor acknowledged
- sponsor confirmed

# [Vault: Redundant notHalted modifier in depositMultiple()](https://github.com/code-423n4/2021-09-yaxis-findings/issues/70) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `notHalted` modifier in `depositMultiple()` is redundant because it is checked (multiple times) by the underlying function call to `deposit()`.

Further optimizations may be done to implement an internal `_deposit()` function that will be called by both `deposit()` and `depositMultiple()` so that `notHalted` is only checked once.

### Recommended Mitigation Steps

Remove the `notHalted` modifier in `depositMultiple()`.

