## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Mark ConvexStakingWrapper.addRewards as External](https://github.com/code-423n4/2022-02-concur-findings/issues/2) 

# Handle

Heartless


# Vulnerability details

## Impact
external visibility uses less gas than public visibility. addRewards is never called internally in this project so does not need public visibility. addRewards was public in the original source from convex because they called addRewards() internally in the initialize() function, which ConvexStakingWrapper does not have.

## Proof of Concept
Line 93 in ConvexStakingWrapper.sol

## Tools Used

## Recommended Mitigation Steps
Change addRewards visibility to external.

