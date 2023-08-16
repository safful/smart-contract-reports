## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [sYETIToken does not emit Approval event in `transferFrom`](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/205) 

# Handle

cmichel


# Vulnerability details

The `sYETIToken.transferFrom` function does not emit a new `Approval` event when decreasing the allowance.
Most ERC20 implementations, like OpenZeppelin's, emit this event when the `allowance` is decreased.

## Impact
Off-chain scripts and frontends will not correctly track the `allowance`s of users when listening to the `Approval` event.
This can lead to failed transactions as a higher approval is assumed than it actually is.

## Recommended Mitigation Steps
Emit the `Approval` event also in `transferFrom` if the approval is decreased.


