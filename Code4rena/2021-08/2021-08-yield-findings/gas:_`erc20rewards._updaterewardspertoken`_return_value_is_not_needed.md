## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- ERC20Rewards

# [Gas: `ERC20Rewards._updateRewardsPerToken` return value is not needed](https://github.com/code-423n4/2021-08-yield-findings/issues/34) 

# Handle

cmichel


# Vulnerability details

The return value is never used and should be removed to save some gas.

