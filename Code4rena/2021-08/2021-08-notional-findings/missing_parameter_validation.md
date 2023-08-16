## Tags

- bug
- duplicate
- 0 (Non-critical)
- sponsor confirmed

# [Missing parameter validation](https://github.com/code-423n4/2021-08-notional-findings/issues/57) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

Some parameters of functions are not checked for invalid values:
- `PauseRouter.constructor`: addresses can be zero or not a contract
- `CompoundToNotionalV2.constructor`: addresses can be zero or not a contract


## Impact
Wrong user input or wallets defaulting to the zero addresses for a missing input can lead to the contract needing to redeploy or wasted gas.

## Recommended Mitigation Steps
Validate the parameters.

