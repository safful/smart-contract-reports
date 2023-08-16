## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Unused imported interface in LendingPair](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/128) 

# Handle

0xsanson


# Vulnerability details

## Impact
The './interfaces/IInterestRateModel.sol' imported in LendingPair.sol isn't actually used and can be removed

## Proof of Concept
https://github.com/code-423n4/2021-07-wildcredit/blob/main/contracts/LendingPair.sol#L13

## Tools Used
Manual Analysis

## Recommended Mitigation Steps
Remove the import line.

