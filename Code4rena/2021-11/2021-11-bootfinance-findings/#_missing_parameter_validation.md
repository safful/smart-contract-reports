## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [# Missing parameter validation](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/209) 

# Handle

cmichel


# Vulnerability details

Some parameters of functions are not checked for invalid values:
- `Swap.setAdminFee`: The `newAdminFee` should be validated the same way as in the constructor
- `Swap.setSwapFee`: The `newSwapFee` should be validated the same way as in the constructor
- `Swap.setDefaultWithdrawFee`: The `newWithdrawFee` should be validated the same way as in the constructor

## Impact
Wrong user input or wallets defaulting to the zero addresses for a missing input can lead to the contract needing to redeploy or wasted gas.

## Recommended Mitigation Steps
Validate the parameters.

