## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Use immutable](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/132) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Gas saving.

## Proof of Concept
The variable `yetiToken` inside the `LockupContract` contract is never modified, so it's better to use immutable to avoid storage access.

## Tools Used
Gas saving

## Recommended Mitigation Steps
Use immutable

