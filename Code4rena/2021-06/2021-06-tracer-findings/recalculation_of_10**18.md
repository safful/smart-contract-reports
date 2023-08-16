## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [recalculation of 10**18](https://github.com/code-423n4/2021-06-tracer-findings/issues/50) 

# Handle

pauliax


# Vulnerability details

## Impact
Insurance function drainPool calculates 10**18 many times. To reduce the number of calculations and save gas, this number can be extracted as a constant variable and used everywhere where necessary.

## Recommended Mitigation Steps
Extract 10**18 as a constant.

