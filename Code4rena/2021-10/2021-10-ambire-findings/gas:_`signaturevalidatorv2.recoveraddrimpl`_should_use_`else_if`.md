## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Gas: `SignatureValidatorV2.recoverAddrImpl` should use `else if`](https://github.com/code-423n4/2021-10-ambire-findings/issues/42) 

# Handle

cmichel


# Vulnerability details

The `SignatureValidatorV2.recoverAddrImpl` function currently uses three `if (mode == *)` checks but the modes are all distinct enum values and therefore an `else if` can be used.
This is more efficient because if the first branch is already matched, there's no need to check the `mode` against the remaining values anymore.

