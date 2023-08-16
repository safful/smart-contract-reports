## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [CollSurplusPool doesn't verify that the passed `_whitelistAddress` is an actual contract addres](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/230) 

# Handle

Ruhum


# Vulnerability details

## Impact
All the other passed variables are checked. Only `_whitelistAddress` is ignored. This allows passing a zero function which would break the functionality.

## Proof of Concept
https://github.com/code-423n4/2021-12-yetifinance/blob/main/packages/contracts/contracts/CollSurplusPool.sol#L51-L54

## Tools Used
none

## Recommended Mitigation Steps
add `checkContract(_whitelistAddress)`

