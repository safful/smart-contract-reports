## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Functions returning boolean](https://github.com/code-423n4/2021-09-swivel-findings/issues/149) 

# Handle

pauliax


# Vulnerability details

## Impact
It is unclear why there are many functions that always return a boolean value of true. While this may be your agreed practice that you try to follow, it also incurs more gas consumption as the caller needs to receive and check these returned values.

## Recommended Mitigation Steps
If you want to optimize for gas, consider dropping return values for functions that actually do not need them.

