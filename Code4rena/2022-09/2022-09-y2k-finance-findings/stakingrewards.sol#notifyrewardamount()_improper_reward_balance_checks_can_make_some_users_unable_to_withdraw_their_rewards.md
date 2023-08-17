## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [StakingRewards.sol#notifyRewardAmount() Improper reward balance checks can make some users unable to withdraw their rewards](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/50) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L201-L205


# Vulnerability details

## Impact
Similar to https://github.com/code-423n4/2022-02-concur-findings/issues/209
```
        uint256 balance = rewardsToken.balanceOf(address(this));
        require(
            rewardRate <= balance.div(rewardsDuration),
            "Provided reward too high"
        );
```
In the current implementation, the contract only checks if balanceOf rewardsToken is greater than or equal to the future rewards.

However, under normal circumstances, since users can not withdraw all their rewards in time, the balance in the contract contains rewards that belong to the users but have not been withdrawn yet. This means the current checks can not be sufficient enough to make sure the contract has enough amount of rewardsToken.

As a result, if the rewardsDistribution mistakenly notifyRewardAmount with a larger amount, the contract may end up in a wrong state that makes some users unable to claim their rewards.

Given:

+ rewardsDuration = 7 days;
1. Alice stakes 1,000 stakingToken;
2. rewardsDistribution sends 100 rewardsToken to the contract;
3. rewardsDistribution calls notifyRewardAmount() with amount = 100;
4. 7 days later, Alice calls earned() and it returns 100 rewardsToken, but Alice choose not to getReward() for now;
5. rewardsDistribution calls notifyRewardAmount() with amount = 100 without send any fund to contract, the tx will succees;
6. 7 days later, Alice calls earned() 200 rewardsToken, when Alice tries to call getReward(), the transaction will fail due to insufficient balance of rewardsToken.

Expected Results:

The tx in step 5 should revert.
## Proof of Concept

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L201-L205
## Tools Used
None
## Recommended Mitigation Steps
Consider changing the function notifyRewardAmount to addRward and use transferFrom to transfer rewardsToken into the contract:
```
function addRward(uint256 reward)
    external
    updateReward(address(0))
{
    require(
        msg.sender == rewardsDistribution,
        "Caller is not RewardsDistribution contract"
    );

    if (block.timestamp >= periodFinish) {
        rewardRate = reward / rewardsDuration;
    } else {
        uint256 remaining = periodFinish - block.timestamp;
        uint256 leftover = remaining * rewardRate;
        rewardRate = (reward + leftover) / rewardsDuration;
    }

    rewardsToken.safeTransferFrom(msg.sender, address(this), reward);

    lastUpdateTime = block.timestamp;
    periodFinish = block.timestamp + rewardsDuration;
    emit RewardAdded(reward);
}
```