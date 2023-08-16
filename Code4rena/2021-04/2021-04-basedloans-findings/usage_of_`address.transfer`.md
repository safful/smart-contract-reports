## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Usage of `address.transfer`](https://github.com/code-423n4/2021-04-basedloans-findings/issues/31) 

# Handle

@cmichelio


# Vulnerability details


## Vulnerability Details

The `transfer` function is used in `Maximillion.sol` to send ETH to an account.

## Impact

It is performed with a fixed amount of GAS and might fail if GAS costs change in the future or if a smart contract's fallback function handler is complex.

## Recommended Mitigation Steps

Consider using the lower-level `.call{value: value}` instead and checking its success return value.

