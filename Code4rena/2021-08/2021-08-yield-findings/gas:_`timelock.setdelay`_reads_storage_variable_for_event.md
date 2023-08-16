## Tags

- bug
- duplicate
- G (Gas Optimization)
- sponsor confirmed
- Timelock

# [Gas: `TimeLock.setDelay` reads storage variable for event](https://github.com/code-423n4/2021-08-yield-findings/issues/33) 

# Handle

cmichel


# Vulnerability details

`TimeLock.setDelay` reads storage variable for event which produces an `SLOAD`. It should use `emit DelaySet(_delay)` instead of `emit DelaySet(delay)`

