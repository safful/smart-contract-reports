## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing parameter validation](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/41) 

# Handle

cmichel


# Vulnerability details

Some parameters of functions are not checked for invalid values:
- `WrappedIbbtcEth.initialize`: The parameters could be checked to be non-zero or even if they're contracts implementing the desired interfaces.

## Impact
Wrong user input or wallets defaulting to the zero addresses for a missing input can lead to the contract needing to redeploy or wasted gas.

## Recommended Mitigation Steps
Validate the parameters.

