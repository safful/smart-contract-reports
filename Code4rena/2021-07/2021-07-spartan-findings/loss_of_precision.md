## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Loss of precision](https://github.com/code-423n4/2021-07-spartan-findings/issues/224) 

# Handle

0xsanson


# Vulnerability details

## Impact
In Router.sol, there's a loss of precision that can be corrected by shifting the operations.

## Proof of Concept
https://github.com/code-423n4/2021-07-spartan/blob/main/contracts/Router.sol#L274

## Tools Used
editor

## Recommended Mitigation Steps
Consider rewriting L274-275 with `uint numerator = (_fees * reserve) / eraLength / maxTrades;`.

