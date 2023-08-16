## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: `incident.payoutNumerator` is used only once. It shouldn't be stored in a variable.](https://github.com/code-423n4/2022-01-insure-findings/issues/349) 

# Handle

Dravee


# Vulnerability details

## Impact
Increased gas cost (1 MSTORE and 1 MLOAD)

## Proof of Concept
https://github.com/code-423n4/2022-01-insure/blob/main/contracts/PoolTemplate.sol#L552
There's no readability or gas gain from copying `incident.payoutNumerator` to a variable as it's used only once in the method.

## Tools Used
VS Code

## Recommended Mitigation Steps
Do not store this data in a variable

