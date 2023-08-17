## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-05

# [Incorrect calculation of the remaining updatedRewards leads to possible underflow error](https://github.com/code-423n4/2023-05-ajna-findings/issues/440) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L549
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L725


# Vulnerability details



## Impact

`RewardsManage.sol` keeps track of the **total number of rewards collected per epoch** for all pools:

```c
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol
73:    /// @dev `epoch => rewards claimed` mapping.
74:    mapping(uint256 => uint256) public override rewardsClaimed;
75:    /// @dev `epoch => update bucket rate rewards claimed` mapping.
76:    mapping(uint256 => uint256) public override updateRewardsClaimed;
```

And the `rewardsCap` calculation when calculating the reward applies only to the pool, which leads to a situation when the condition is fulfilled `rewardsClaimedInEpoch + updatedRewards_ >= rewardsCap`, **But `rewardsCap` is less than `rewardsClaimedInEpoch`**:

```diff
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

-543:         uint256 rewardsCapped = Maths.wmul(REWARD_CAP, totalBurnedInPeriod);
545:         // Check rewards claimed - check that less than 80% of the tokens for a given burn event have been claimed.
-546:         if (rewardsClaimedInEpoch_ + newRewards_ > rewardsCapped) {
548:             // set claim reward to difference between cap and reward
-549:             newRewards_ = rewardsCapped - rewardsClaimedInEpoch_; // @audit rewardsCapped can be less then  rewardsClaimedInEpoch_
550:         }

719:         uint256 rewardsCap            = Maths.wmul(UPDATE_CAP, totalBurned); // @audit in one pool
-720:        uint256 rewardsClaimedInEpoch = updateRewardsClaimed[curBurnEpoch];
722:         // update total tokens claimed for updating bucket exchange rates tracker
723:         if (rewardsClaimedInEpoch + updatedRewards_ >= rewardsCap) {
724:              // if update reward is greater than cap, set to remaining difference
-725:             updatedRewards_ = rewardsCap - rewardsClaimedInEpoch; // @audit rewardsCap can be less then rewardsClaimedInEpoch
726:         }
728:         // accumulate the full amount of additional rewards
-729:        updateRewardsClaimed[curBurnEpoch] += updatedRewards_; // @audit increase per epoch
```

which causes an `underflow` erorr in the result `updatedRewards_ = rewardsCap - rewardsClaimedInEpoch` where `rewardsCap < rewardsClaimedInEpoch`, this error **leads to a transaction fail**, which will further temporarily/permanently block actions with NFT as `unstake/claimRewards` for pools in which `rewardsCap` will fail less than the total `rewardsClaimedInEpoch`

We have 2 instances of this problem::

1. during the call `_calculateNewRewards`
2. during the call `_updateBucketExchangeRates`

a failure in any of these will result in users of certain pools being unable to withdraw their NFTs as well as the reward

## Proof of Concept

Let's take a closer look at the problem and why this is possible:

1. We have a general calculation of rewards taken per epoch:

```javascript
File: ajna-core\src\RewardsManager.sol

71:     /// @dev `epoch => rewards claimed` mapping.
72:     mapping(uint256 => uint256) public override rewardsClaimed;
73:     /// @dev `epoch => update bucket rate rewards claimed` mapping.
74:     mapping(uint256 => uint256) public override updateRewardsClaimed;
```

2. The state is updated for the epoch by the amount calculated for each pool:

```javascript
File: ajna-core\src\RewardsManager.sol

_calculateAndClaimRewards
396:         for (uint256 epoch = lastClaimedEpoch; epoch < epochToClaim_; ) {

410:             // update epoch token claim trackers
411:             rewardsClaimed[epoch]           += nextEpochRewards;

413:         }

_updateBucketExchangeRates
676:         uint256 curBurnEpoch = IPool(pool_).currentBurnEpoch();

728:                 // accumulate the full amount of additional rewards
729:                 updateRewardsClaimed[curBurnEpoch] += updatedRewards_;
```

3. At the time of calculation of the reward for the update:

```diff
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

526:         (
527:             ,
528:             // total interest accumulated by the pool over the claim period
+529:             uint256 totalBurnedInPeriod,
530:             // total tokens burned over the claim period
531:             uint256 totalInterestEarnedInPeriod
532:         ) = _getPoolAccumulators(ajnaPool_, nextEpoch_, epoch_);
533:
534:         // calculate rewards earned
                ...
542:
+543:         uint256 rewardsCapped = Maths.wmul(REWARD_CAP, totalBurnedInPeriod);
544:
545:         // Check rewards claimed - check that less than 80% of the tokens for a given burn event have been claimed.
546:         if (rewardsClaimedInEpoch_ + newRewards_ > rewardsCapped) {
547:
548:             // set claim reward to difference between cap and reward
+549:             newRewards_ = rewardsCapped - rewardsClaimedInEpoch_; // @audit
550:         }
```

We have a situation where `rewardsClaimedInEpoch_` has been updated by other pools to something like `100e18`,
and `rewardsCapped` for the other pool was `30e18`, resulting in:
` rewardsClaimedInEpoch_ + newRewards_ > rewardsCapped`
and of course we catch the underflow at the time of calculating the remainder, `30e18 - 100e18`, since there is no remainder
`newRewards_ = rewardsCapped - rewardsClaimedInEpoch_`


To check the problem, you need to raise `rewardsClaimedInEpoch_` more than the `rewardsCap` of a certain pool, with the help of other pools,

`rewardsCap` is a percentage of burned tokens in the pool... so it's possible

## Tools Used

- Manual review
- Foundry

## Recommended Mitigation Steps

- Add additional requirements that if `rewardsClaimedInEpoch > rewardsCap` that `updatedRewards_` should be zero, not need calculate remaining difference



## Assessed type

Invalid Validation