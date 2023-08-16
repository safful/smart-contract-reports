## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Emit an event in setKeeper()](https://github.com/code-423n4/2022-01-insure-findings/issues/132) 

# Handle

p4st13r4


# Vulnerability details

## Impact

The `setKeeper()` function is operated only by the owner, and should emit an event when the keeper is set for the first time and/or changes

## Proof of Concept

[https://github.com/code-423n4/2022-01-insure/blob/main/contracts/Vault.sol#L502](https://github.com/code-423n4/2022-01-insure/blob/main/contracts/Vault.sol#L502)

## Tools Used

Editor

## Recommended Mitigation Steps

Add `emit KeeperChanged(address)` after changing the keeper

