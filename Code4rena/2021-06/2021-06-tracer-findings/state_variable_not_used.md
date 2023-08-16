## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [State variable not used](https://github.com/code-423n4/2021-06-tracer-findings/issues/122) 

# Handle

0xsanson


# Vulnerability details

## Impact
State variable perpsFactory is not used in the Insurance contract.

## Proof of Concept
https://github.com/code-423n4/2021-06-tracer/blob/main/src/contracts/Insurance.sol#L18

## Tools Used
Manual analysis

## Recommended Mitigation Steps
Just delete it.

