## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- sponsor acknowledged
- selected for report
- M-07

# [ Fees charged from entire theoretical pledge amount instead of actual pledge amount](https://github.com/code-423n4/2022-10-paladin-findings/issues/235) 

# Lines of code

https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L328


# Vulnerability details

## Description

Paladin receives a 5% cut from Boost purchases, as documented on the [website](https://paladin.vote/#/) 

"Warden takes a 5% fee on Boost purchases, and 5% on Quest incentives. However, there are various pricing tiers for Quest creators. Contact the Paladin team for more info."

Here's how fee calculation looks at  `createPledge` function:
```
vars.totalRewardAmount = (rewardPerVote * vars.votesDifference * vars.duration) / UNIT;
vars.feeAmount = (vars.totalRewardAmount * protocalFeeRatio) / MAX_PCT ;
if(vars.totalRewardAmount > maxTotalRewardAmount) revert Errors.IncorrectMaxTotalRewardAmount();
if(vars.feeAmount > maxFeeAmount) revert Errors.IncorrectMaxFeeAmount();
// Pull all the rewards in this contract
IERC20(rewardToken).safeTransferFrom(creator, address(this), vars.totalRewardAmount);
// And transfer the fees from the Pledge creator to the Chest contract
IERC20(rewardToken).safeTransferFrom(creator, chestAddress, vars.feeAmount);
```

The issue is that the fee is taken up front, assuming `totalRewardAmount` will actually be rewarded by the pledge. In practice, the rewards actually utilized can be anywhere from zero to `totalRewardAmount`. Indeed, reward will only be `totalRewardAmount` if, in the entire period from pledge creation to pledge expiry, the desired targetVotes will be fulfilled, which is extremly unlikely. 

As a result, if pledge expires with no pledgers, protocol will still take 5%. This behavior is both unfair and against the docs, as it's not "Paladin receives a 5% cut from Boost purchases".

## Impact

Paladin fee collection assumes pledges will be matched immediately and fully, which is not realistic. Therefore far too much fees are collected at user's expense.

## Proof of Concept

1. Bob creates a pledge, with target = 200, current balance = 100, rewardVotes = 10, remaining time = 1 week.
2. Protocol collects (200 - 100) * 10 * WEEK_SECONDS * 5% fees
3. A week passed, rewards were not attractive enough to bring pledgers.
4. After expiry, Bob calls retrievePledgeRewards() and gets 100 * 10 * WEEK_SECONDS back, but 5% of the fees still went to chestAddress.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Fee collection should be done after the pledge completes, in one of the close functions or in a newly created pull function for owner to collect fees. Otherwise, it is a completely unfair system.