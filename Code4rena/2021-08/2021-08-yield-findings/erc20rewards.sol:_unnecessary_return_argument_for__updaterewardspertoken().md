## Tags

- bug
- duplicate
- G (Gas Optimization)
- sponsor confirmed
- ERC20Rewards

# [ERC20Rewards.sol: Unnecessary return argument for _updateRewardsPerToken()](https://github.com/code-423n4/2021-08-yield-findings/issues/60) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `uint128` output by `_updateRewardsPerToken()` isn't used by any function. Furthermore, L107 returns a wrong value.

`if (_totalSupply == 0 || block.timestamp.u32() < rewardsPeriod.start) return 0;`

It should return `rewardsPerToken_.accumulated` instead, because in the case of a new rewards schedule being set, `rewardsPeriod.start` is updated to a new timestamp. Hence, the current accumulated value of all previous reward schedules thus far should be returned.

### Recommended Mitigation Steps

Since the return value isn't used anywhere, remove it.

