## Tags

- bug
- 2 (Med Risk)
- sponsor acknowledged
- selected for report

# [StakingRewards: Significant loss of precision possible](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/225) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/bca5080635370424a9fe21fe1aded98345d1f723/src/rewards/StakingRewards.sol#L194


# Vulnerability details

## Impact
In `notifyRewardAmount`, the reward rate per second is calculated. This calculation rounds down, which can lead to situations where significantly less rewards are paid out to stakers, because the effect of the rounding is multiplied by the duration.

## Proof Of Concept
Let's say we have a `rewardsDuration` of 4 years, i.e. 126144000 seconds. We assume the `rewardRate` is currently ß and `notifyRewardAmount` is called with the reward amount 252287999. Because the calculation rounds down, `rewardRate` will be 1.
After the 4 years, the user have received 126144000 reward tokens. However, 126143999 (i.e., almost 50%) of the reward tokens that were intended to be distributed to the stakers were not distributed, resulting in monetary loss for all stakers.

## Recommended Mitigation Steps
You could accumulate the differences that occur due to rounding and let the users claim them in the end according to their shares.