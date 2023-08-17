## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- fix security (sponsor)
- M-15

# [wrong reward distribution between early and late depositors because of the late syncRewards() call in the cycle, syncReward() logic should be executed in each withdraw or deposits (without reverting)](https://github.com/code-423n4/2022-12-gogopool-findings/issues/478) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L88-L109
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L166-L178
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/tokens/TokenggAVAX.sol#L180-L189


# Vulnerability details

## Impact
Function `syncRewards()` distributes rewards to TokenggAVAX holders, it linearly distribute cycle's rewards from `block.timestamp` to the cycle end time which is next multiple of the `rewardsCycleLength` (the end time of the cycle is defined and the real duration of the cycle changes). when a cycle ends `syncRewards()` should be called so the next cycle starts but if `syncRewards()` doesn't get called fast, then users depositing or withdrawing funds before call to `syncRewards()` would lose their rewards and those rewards would go to users who deposited funds after `syncRewards()` call. contract should try to start the next cycle whenever deposit or withdraw happens to make sure rewards are distributed fairly between users.

## Proof of Concept
This is `syncRewards()` code:
```
	function syncRewards() public {
		uint32 timestamp = block.timestamp.safeCastTo32();

		if (timestamp < rewardsCycleEnd) {
			revert SyncError();
		}

		uint192 lastRewardsAmt_ = lastRewardsAmt;
		uint256 totalReleasedAssets_ = totalReleasedAssets;
		uint256 stakingTotalAssets_ = stakingTotalAssets;

		uint256 nextRewardsAmt = (asset.balanceOf(address(this)) + stakingTotalAssets_) - totalReleasedAssets_ - lastRewardsAmt_;

		// Ensure nextRewardsCycleEnd will be evenly divisible by `rewardsCycleLength`.
		uint32 nextRewardsCycleEnd = ((timestamp + rewardsCycleLength) / rewardsCycleLength) * rewardsCycleLength;

		lastRewardsAmt = nextRewardsAmt.safeCastTo192();
		lastSync = timestamp;
		rewardsCycleEnd = nextRewardsCycleEnd;
		totalReleasedAssets = totalReleasedAssets_ + lastRewardsAmt_;
		emit NewRewardsCycle(nextRewardsCycleEnd, nextRewardsAmt);
	}
```
As you can see whenever this function is called it starts the new cycle and sets the end of the cycle to the next multiple of the `rewardsCycleLength` and it release the rewards linearly between current timestamp and cycle end time. so if `syncRewards()` get called near to multiple of the `rewardsCycleLength` then rewards would be distributed with higher speed in less time. the problem is that users depositing funds before call `syncRewards()` won't receive new cycles rewards and early depositing won't get considered in reward distribution if deposits happen before `syncRewards()` call and if a user withdraws his funds before the `syncRewards()` call then he receive no rewards.
imagine this scenario:
1. `rewardsCycleLength` is 10 days and the rewards for the next cycle is `100` AVAX.
2. the last cycle has been ended and user1 has `10000` AVAX deposited and has 50% of the pool shares.
3. `syncRewards()` don't get called for 8 days.
3. users1 withdraws his funds receive `10000` AVAX even so he deposits for 8 days in the current cycle.
4. users2 deposit `1000` AVAX and get 10% of pool shares and the user2 would call `syncRewards()` and contract would start distributing `100` avax as reward.
5. after 2 days cycle would finish and user2 would receive `100 * 10% = 10` AVAX as rewards for his `1000` AVAX deposit for 2 days but user1 had `10000` AVAX for 8 days and would receive 0 rewards.

so rewards won't distribute fairly between depositors across the time and any user interacting with contract before the `syncRewards()` call can lose his rewards. contract won't consider deposit amounts and duration before `syncRewards()` call and it won't make sure that `syncRewards()` logic would be executed as early as possible with deposit or withdraw calls when a cycle ends.

## Tools Used
VIM

## Recommended Mitigation Steps
one way to solve this is to call `syncRewards()` logic in each deposit or withdraw and make sure that cycles start as early as possible (the revert "SyncError()" in the `syncRewards()` should be removed for this).