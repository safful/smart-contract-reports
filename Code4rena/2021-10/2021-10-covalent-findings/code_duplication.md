## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Code duplication](https://github.com/code-423n4/2021-10-covalent-findings/issues/46) 

# Handle

WatchPug


# Vulnerability details

Duplicated or logically equivalent code can be hard to maintain. Avoiding code duplication is recommended when feasible.

For example, most of the business logic in `redeemAllRewards()` and `redeemRewards()` is the same.

Consider calculating the amount of the total rewards in `redeemAllRewards()` and call `redeemRewards()` with the amount to reduce code duplication.

