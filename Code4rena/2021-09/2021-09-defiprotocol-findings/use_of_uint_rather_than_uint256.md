## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Use of uint rather than uint256](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/69) 

# Handle

loop


# Vulnerability details

In basket.sol there is one use of `uint` rather than `uint256`, which is used in the rest of the codebase.

## Impact
No real impact considering `uint` functions as a `uint256`.

## Proof of Concept
Basket.sol - line 60:
`for (uint i = 0; i < length; i++) {`

