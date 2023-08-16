## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Immutable variables](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/47) 

# Handle

pauliax


# Vulnerability details

## Impact
'immutable' greatly reduces gas costs. There are variables that do not change so they can be marked as immutable to greatly improve the gas costs. 
Examples of such variables are: 
yieldSource in YieldSourcePrizePool. 
prizePool in PrizeSplitStrategy. 
controller in ControlledToken.

## Recommended Mitigation Steps
Consider applying immutable keyword to the variables that are assigned only once.

