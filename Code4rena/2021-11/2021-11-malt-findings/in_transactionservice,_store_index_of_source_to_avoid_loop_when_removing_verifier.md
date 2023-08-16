## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [In TransactionService, store index of source to avoid loop when removing verifier](https://github.com/code-423n4/2021-11-malt-findings/issues/199) 

# Handle

xYrYuYx


# Vulnerability details

## Impact
In removeVerifier function, it loop until last index - 1 to find source index.
If you added many verifiers, then the gas cost of removeVerifier will be very high, and it can be reverted due to gas limit as well.


## Tools Used
Manual

## Recommended Mitigation Steps
Store index of address in addVerifier function, and remove loop in removeVerifier, and use stored index.

