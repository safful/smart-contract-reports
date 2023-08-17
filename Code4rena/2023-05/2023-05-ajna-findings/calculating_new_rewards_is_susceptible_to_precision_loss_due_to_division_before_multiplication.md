## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-05

# [Calculating new rewards is susceptible to precision loss due to division before multiplication](https://github.com/code-423n4/2023-05-ajna-findings/issues/367) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L519-L541


# Vulnerability details

This issue is similar to https://github.com/ajna-finance/audits/blob/main/sherlock/Contest1.md#issue-m-7-calculating-new-rewards-is-susceptible-to-precision-loss-due-to-division-before-multiplication which is not fixed properly. Still, the final multiplication is being performed after the division.

## Impact
Rewards may be lost (0) due to division before multiplication precision issues.

## Proof of Concept
The `RewardsManager._calculateNewRewards` function calculates the new rewards for a staker by first multiplying `interestEarned_` by `totalBurnedInPeriod` and then dividing by `totalInterestEarnedInPeriod` and then again multiplying by `REWARD_FACTOR`
Since the division is being performed before the final multiplication, this can lead to precision loss.

```solidity
    function _calculateNewRewards(
        address ajnaPool_,
        uint256 interestEarned_,
        uint256 nextEpoch_,
        uint256 epoch_,
        uint256 rewardsClaimedInEpoch_
    ) internal view returns (uint256 newRewards_) {
        (
            ,
            // total interest accumulated by the pool over the claim period
            uint256 totalBurnedInPeriod,
            // total tokens burned over the claim period
            uint256 totalInterestEarnedInPeriod
        ) = _getPoolAccumulators(ajnaPool_, nextEpoch_, epoch_);

        // calculate rewards earned 
        newRewards_ = totalInterestEarnedInPeriod == 0 ? 0 : Maths.wmul(
            REWARD_FACTOR,
            Maths.wdiv(
                Maths.wmul(interestEarned_, totalBurnedInPeriod), 
                totalInterestEarnedInPeriod
            )
        );
```

## Recommended Mitigation Steps
All the multiplication should be performed in step 1 and then division at the end.


## Assessed type

Math