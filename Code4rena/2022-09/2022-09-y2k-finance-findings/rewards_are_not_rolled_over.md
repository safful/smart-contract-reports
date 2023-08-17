## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [Rewards are not rolled over](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/93) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/rewards/StakingRewards.sol#L183


# Vulnerability details

## Impact
If there is no deposit for sometime in start then reward for those period is never used

## Proof of Concept
1. Admin has added reward which made reward rate as 10 reward per second using notifyRewardAmount function

```
rewardRate = reward.div(rewardsDuration);
```

2. For initial 10 seconds there were no deposits which means total supply was 0
3. So no reward were distributed for initial 10 seconds and reward for this duration which is 10*10=100 will remain in contract
4. Since on notifying contract of new rewards, these stuck rewards are not considered so these 100 rewards will remain in contract with no usage

## Recommended Mitigation Steps
On very first deposit better to have (block.timestamp-startTime) * rewardRate amount of reward being marked unused which can be used in next notifyrewardamount