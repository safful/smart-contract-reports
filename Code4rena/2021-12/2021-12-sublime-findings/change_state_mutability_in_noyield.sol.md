## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Change state mutability in NoYield.sol](https://github.com/code-423n4/2021-12-sublime-findings/issues/165) 

# Handle

p4st13r4


# Vulnerability details

## Impact
The `liquidityToken` function in `NoYield.sol` can have its state mutability changed to `pure` instead of `view`. This has no other effect than suppressing compiler warnings, but may help the compiler optimize this function in the future

## Proof of Concept

## Tools Used

## Recommended Mitigation Steps
Change state mutability of `liquidityToken` to `pure`

