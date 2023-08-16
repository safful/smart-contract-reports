## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [nTokenERC20Proxy emits events even when not success](https://github.com/code-423n4/2021-08-notional-findings/issues/72) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `nTokenERC20Proxy` functions emit events all the time, even if the return value from the inner call returns `false` indicating an unsuccessful action.

## Impact
An off-chain script scanning for `Transfer` or `Approval` events can be tricked into believing that an unsuccessful transfer was indeed successful.
This happens in the `approve`, `transfer` and `transferFrom` functions.

## Recommended Mitigation Steps
Only emit evens on `success`.

