## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-12

# [Rewards for initial period can be lost in all of the synthetix derivative contracts](https://github.com/code-423n4/2023-06-lybra-findings/issues/484) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L132-L150
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L227-L240
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/EUSDMiningIncentives.sol#L226-L242


# Vulnerability details

## Impact
Rewards in the synthetix derivative contracts (EUSDMinningIncentives.sol, ProtocolRewardsPool.sol and stakerRewardsV2Pool.sol) contract are initiated when the owner calls the notifyRewardAmount. This function calculates the reward rate per second and also records the start of the reward period. This has an edge case where rewards are not counted for the initial period of time until there is at least one participant. 

## Proof of Concept

Look at the code for the stakerrewardV2Pool.sol, and other files have somewhat similar logic too, derived from the synthetix:

```solidity
    function notifyRewardAmount(uint256 _amount) external onlyOwner updateReward(address(0)) {
        if (block.timestamp >= finishAt) {
            rewardRatio = _amount / duration;
        } else {
            uint256 remainingRewards = (finishAt - block.timestamp) * rewardRatio;
            rewardRatio = (_amount + remainingRewards) / duration;
        }

        require(rewardRatio > 0, "reward ratio = 0");

        finishAt = block.timestamp + duration;
        updatedAt = block.timestamp;
        emit NotifyRewardChanged(_amount, block.timestamp);
    }

    function _min(uint256 x, uint256 y) private pure returns (uint256) {
        return x <= y ? x : y;
    }
}
```
The intention here is to calculate how many tokens should be rewarded by unit of time (second) and record the span of time for the reward cycle. However, this has an edge case where rewards are not counted for the initial period of time until there is at least one participant (in this case, a holder of BathTokens). During this initial period of time, the reward rate will still apply but as there isn't any participant, then no one will be able to claim these rewards and these rewards will be lost and stuck in the system.



This is a known vulnerability that have been covered before, The following reports can be used as a reference for the described issue:

[0xMacro Blog - Synthetix Vulnerability](https://0xmacro.com/blog/synthetix-staking-rewards-issue-inefficient-reward-distribution/)

[Same vulnerability in y2k report](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/93)

As described by the 0xmacro blogpost, this can play as following:

Let’s consider that you have a StakingRewards contract with a reward duration of one month seconds (2592000):

Block N             Timestamp = X

You call notifyRewardAmount() with a reward of one month seconds (2592000) only. The intention is for a period of a month, 1 reward token per second should be distributed to stakers.

State :

rewardRate = 1

periodFinish = X +**2592000**

Block M          Timestamp = X + Y

Y time has passed, and the first staker stakes some amount:

1.stake()
2. updateReward
rewardPerTokenStored = 0
lastUpdateTime = X + Y

Hence, for this staker, the clock has started from X+Y, and he or she will accumulate rewards from this point.

Please note, though, that the periodFinish is X + rewardsDuration, not X + Y + rewardsDuration. Therefore, the contract will only distribute rewards until X + rewardsDuration, losing  Y * rewardRate => Y * 1  inside of the contract, as rewardRate = 1 (if we consider the above example).

Now, if we consider delay(Y) to be 30 minutes, then:

Only 2592000-1800= 2590200 tokens will be distributed, and these 1800 tokens will remain unused in the contract until the next cycle of notifyRewardAmount().

Same issue was reported in the rubicon v2 contest on code4rena, that can be looked for more reference.

## Tools Used
Manual Review

## Recommended Mitigation Steps
A possible solution to the issue would be to set the start and end time for the current reward cycle when the first participant joins the reward program, i.e. when the total supply is greater than zero, instead of starting the process in the notifyRewardAmount.


## Assessed type

Other