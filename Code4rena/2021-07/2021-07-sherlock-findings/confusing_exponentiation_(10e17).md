## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Confusing exponentiation (10e17)](https://github.com/code-423n4/2021-07-sherlock-findings/issues/129) 

# Handle

0xsanson


# Vulnerability details

## Impact
The value `10e17` can be confusing, since it doesn't clearly appear from where the exponent 17 comes from (people may ctrl+f or grep the code for other instances of it without results). Indeed throughout the code the expression `10**18` is used.

## Proof of Concept
https://github.com/code-423n4/2021-07-sherlock/blob/main/contracts/facets/Payout.sol#L185

## Tools Used
editor

## Recommended Mitigation Steps
Better ways of writing it are `1e18` or `10**18`.

