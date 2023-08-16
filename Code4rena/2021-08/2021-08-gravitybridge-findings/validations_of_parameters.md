## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Validations of parameters](https://github.com/code-423n4/2021-08-gravitybridge-findings/issues/31) 

# Handle

pauliax


# Vulnerability details

## Impact
There are a few validations that could be added to the system:
the constructor could check that _gravityId is not empty. state_powerThreshold should always be greater than 0, otherwise, anyone will be available to execute actions.

## Recommended Mitigation Steps
Consider implementing suggested validations.

