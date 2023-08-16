## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Field bonusMultiplier of struct Position can be removed](https://github.com/code-423n4/2022-01-xdefi-findings/issues/101) 

# Handle

wuwe1


# Vulnerability details

## Proof of Concept

In contract `XDEFIDistribution`, the only use of `bonusMultiplier` is to calculate `units` in `_lock`.

In contract `XDEFIDistributionHelper`, `bonusMultiplier` is used for return value. However, `bonusMultiplier` can be calculated by `units * 100 / depositedXDEFI`.

