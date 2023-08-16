## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Offchain voting can't be renabled in DAO](https://github.com/code-423n4/2021-11-fairside-findings/issues/6) 

# Handle

Ruhum


# Vulnerability details

## Impact
The comment for the `disableOffchainVoting()` function specifies that the feature can be reenabled in the future through a proposal. But, there seems to be no function to do that in the DAO contract.

## Proof of Concept
Function with the comment:
https://github.com/code-423n4/2021-11-fairside/blob/main/contracts/dao/FairSideDAO.sol#L619

No way to reassign the value: `grep offchain`

## Tools Used
Manual Analysis

## Recommended Mitigation Steps
Either remove the comment if the feature is not intended or add a function to reassign the `offchain` and `guardian` state variable

