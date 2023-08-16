## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [use of floating pragma](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/71) 

# Handle

JMukesh


# Vulnerability details

## Impact
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.

## Proof of Concept
most of the contract use floating pragma

## Tools Used
manual review

## Recommended Mitigation Steps
use fixed pragma

