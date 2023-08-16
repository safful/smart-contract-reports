## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Dead code](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/211) 

# Handle

pauliax


# Vulnerability details

## Impact
BLOCK_DECREMENT state variable in Auction is not used anywhere.

## Recommended Mitigation Steps
Consider removing unused variables.

