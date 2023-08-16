## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Wrong revert message](https://github.com/code-423n4/2022-01-xdefi-findings/issues/171) 

# Handle

Czar102


# Vulnerability details

## Impact

Wrong revert messages might lead to confusion.

## Proof of Concept

In line 52 of XDEFIDistribution, the reason for a fail of a reentrant call is `"LOCKED"`. In DeFi, it usually means that contract's functionality is temporarily limited. This is not true in this case.

## Recommended Mitigation Steps

Consider changing the revert string to `"REENTRY_NOT_ALLOWED"`.

