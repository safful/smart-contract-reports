## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [Unbounded iteration over all indexes (2)](https://github.com/code-423n4/2022-01-insure-findings/issues/352) 

# Handle

Dravee


# Vulnerability details

## Impact
The transactions could fail if the array get too big and the transaction would consume more gas than the block limit.
This will then result in a denial of service for the desired functionality and break core functionality.

## Proof of Concept
https://github.com/code-423n4/2022-01-insure/blob/main/contracts/PoolTemplate.sol#L703

## Tools Used
VS Code

## Recommended Mitigation Steps
Keep the array size small.

