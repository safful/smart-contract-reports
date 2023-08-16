## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Synth: Can accidentally burn tokens by sending them to zero](https://github.com/code-423n4/2021-07-spartan-findings/issues/159) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `Synth._transfer` function does not check if `recipient != 0`.
Unlike standard ERC20, tokens can be accidentally burned this way.

## Recommended Mitigation Steps
Prevent user errors by denying transfers to the zero address and forcing them to call `burn` instead.

