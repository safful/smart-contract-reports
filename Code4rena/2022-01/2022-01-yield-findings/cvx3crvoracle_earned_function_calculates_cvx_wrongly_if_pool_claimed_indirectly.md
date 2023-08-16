## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Cvx3CrvOracle earned function calculates cvx wrongly if pool claimed indirectly](https://github.com/code-423n4/2022-01-yield-findings/issues/95) 

# Handle

kenzo


# Vulnerability details

The ConvexStakingWrapper that Yield is based on recently published a fix for `earned` function in case the pool is claimed indirectly.

## Impact
Wrong results might be returned from view function `earned`.

## Proof of Concept
This is the fix for earned: [fix commit](https://github.com/convex-eth/platform/commit/9b9dd72bdb822e7f34f241d620cc1f8388bf7d6a#)

## Recommended Mitigation Steps
Apply fix.

