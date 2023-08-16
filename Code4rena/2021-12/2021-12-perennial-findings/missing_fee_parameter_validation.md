## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Missing fee parameter validation](https://github.com/code-423n4/2021-12-perennial-findings/issues/50) 

# Handle

cmichel


# Vulnerability details

Some fee parameters of functions are not checked for invalid values:
- `Collateral.updateLiquidationFee`: The `newLiquidationFee` should be less than 100%
- `Factory.updateFee`: The `newFee` should be less than 100%

## Recommended Mitigation Steps
Validate the parameters.

