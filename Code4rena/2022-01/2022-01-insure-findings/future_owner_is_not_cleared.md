## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Future owner is not cleared](https://github.com/code-423n4/2022-01-insure-findings/issues/247) 

# Handle

cmichel


# Vulnerability details

The `Ownership.acceptTransferOwnership` function does not reset `_futureOwner` to zero.

## Impact
The future owner can repeatedly accept the governance, emitting an `AcceptNewOwnership` event each time, bloating listeners for this event with unnecessary data.

## Recommended Mitigation Steps
Reset `_futureOwner` to zero in `acceptTransferOwnership`.


