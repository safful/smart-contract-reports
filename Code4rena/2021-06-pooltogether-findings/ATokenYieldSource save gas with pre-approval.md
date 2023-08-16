## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- ATokenYieldSource
- IdleYieldSource

# [ATokenYieldSource save gas with pre-approval](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/93) 

# Handle

cmichel


# Vulnerability details

`ATokenYieldSource` should approve the lending contract once during initialization with the max value.
This saves gas on every `supplyTokenTo/_depositToAave` call as the approval can be removed from there.

