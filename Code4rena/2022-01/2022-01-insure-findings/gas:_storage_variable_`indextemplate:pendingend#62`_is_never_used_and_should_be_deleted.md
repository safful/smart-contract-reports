## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Storage variable `IndexTemplate:pendingEnd#62` is never used and should be deleted](https://github.com/code-423n4/2022-01-insure-findings/issues/68) 

# Handle

Dravee


# Vulnerability details

## Impact
Increased gas cost (1 slot)

## Proof of Concept
IndexTemplate.pendingEnd (contracts/IndexTemplate.sol#62) should be deleted as it's never used by the contract

## Tools Used
Slither

## Recommended Mitigation Steps
Delete the variable `IndexTemplate.pendingEnd`

