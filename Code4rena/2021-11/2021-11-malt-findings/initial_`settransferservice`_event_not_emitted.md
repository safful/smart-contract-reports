## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Initial `SetTransferService` event not emitted](https://github.com/code-423n4/2021-11-malt-findings/issues/249) 

# Handle

cmichel


# Vulnerability details

The initial `SetTransferService` event in `Malt.initialize` is not emitted.

## Impact
Off-chain programs might not correctly track the initial `transferService` variable as the initial event is missing.

## Recommended Mitigation Steps
Emit it in `initialize`.

