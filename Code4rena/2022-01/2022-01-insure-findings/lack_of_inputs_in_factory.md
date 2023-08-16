## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Lack of inputs in Factory](https://github.com/code-423n4/2022-01-insure-findings/issues/120) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Wrong deployment.

## Proof of Concept
The factory contract haven't got any check of `_registry` and `_ownership` and both values must be defined or the logic inside the contract will fault.

## Tools Used
Manual review.

## Recommended Mitigation Steps
It's mandatory to check that the address are not zero or the contract could be wrong deployed.

