## Tags

- bug
- duplicate
- G (Gas Optimization)
- sponsor confirmed
- fixed-in-upstream-repo

# [onlyValidMarket is never used](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/107) 

# Handle

pauliax


# Vulnerability details

## Impact
Dead code: Staker contract has a modifier onlyValidMarket that is not used anywhere. I think you do not use it as you trust that admin and LongShort contract will not pass invalid values. Unused code can be removed to reduce gas costs.

