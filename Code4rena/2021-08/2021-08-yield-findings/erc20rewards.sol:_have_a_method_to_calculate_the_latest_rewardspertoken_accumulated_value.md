## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- ERC20Rewards

# [ERC20Rewards.sol: Have a method to calculate the latest rewardsPerToken accumulated value](https://github.com/code-423n4/2021-08-yield-findings/issues/57) 

# Handle

hickuphh3


# Vulnerability details

### Impact

This would be equivalent to [Unipool's `rewardPerToken()` function](https://github.com/k06a/Unipool/blob/master/contracts/Unipool.sol#L69). Note that `rewardsPerToken.accumulated` only reflects the latest stored accumulated value, but does not account for pending accumulation like Unipool, and is therefore not the same. It possibly might be mistaken to be so, hence the low risk classification.

### Recommended Mitigation Steps

A possible implementation is given below.

```jsx
function latestRewardPerToken() external view returns (uint256) {
	RewardsPerToken memory rewardsPerToken_ = rewardsPerToken;
	if (_totalSupply == 0) return rewardsPerToken_.accumulated;
	uint32 end = earliest(block.timestamp.u32(), rewardsPeriod.end);
	uint256 timeSinceLastUpdated = end - rewardsPerToken_.lastUpdated;
	return rewardsPerToken_.accumulated + 1e18 * timeSinceLastUpdated * rewardsPerToken_.rate / _totalSupply;
}
```

