## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- SushiYieldSource

# [SushiYieldSource save gas with pre-approval](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/94) 

# Handle

cmichel


# Vulnerability details

`SushiYieldSource` should approve the SushiBar once during initialization with the max value.
This saves gas on every `supplyTokenTo` call as the approval can be removed from there.

