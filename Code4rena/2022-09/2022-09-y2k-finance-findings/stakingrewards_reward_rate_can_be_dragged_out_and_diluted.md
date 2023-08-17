## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [StakingRewards reward rate can be dragged out and diluted](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/52) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L183-L195


# Vulnerability details

## Impact
Similar to https://github.com/code-423n4/2022-02-concur-findings/issues/183.
```
        if (block.timestamp >= periodFinish) {
            rewardRate = reward.div(rewardsDuration);
        } else {
            uint256 remaining = periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardRate);
            rewardRate = reward.add(leftover).div(rewardsDuration);
        }
```
The StakingRewards.notifyRewardAmount function receives a reward amount and extends the current reward end time to now + rewardsDuration.
It rebases the currently remaining rewards + the new rewards (reward + leftover) over this new rewardsDuration period.
This can lead to a dilution of the reward rate and rewards being dragged out forever by malicious new reward deposits.
## Proof of Concept
Imagine the current rewardRate is 1000 rewards / rewardsDuration.
20% of the rewardsDuration passed, i.e., now = lastUpdateTime + 20% * rewardsDuration.
A malicious actor notifies the contract with a reward of 0: notifyRewardAmount(0).
Then the new rewardRate = (reward + leftover) / rewardsDuration = (0 + 800) / rewardsDuration = 800 / rewardsDuration.
The rewardRate just dropped by 20%.
This can be repeated infinitely.
After another 20% of reward time passed, they trigger notifyRewardAmount(0) to reduce it by another 20% again:
rewardRate = (0 + 640) / rewardsDuration = 640 / rewardsDuration.

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L183-L195
## Tools Used
None
## Recommended Mitigation Steps
The rewardRate should never decrease by a notifyRewardAmount call.
Consider not extending the reward payouts by rewardsDuration on every call.
periodFinish probably shouldn't change at all, the rewardRate should just increase by rewardRate += reward / (periodFinish - block.timestamp).

Alternatively, consider keeping the rewardRate constant but extend periodFinish time by += reward / rewardRate.