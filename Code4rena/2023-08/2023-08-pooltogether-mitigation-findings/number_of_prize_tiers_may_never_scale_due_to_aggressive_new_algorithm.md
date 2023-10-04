## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- MR-M-16

# [Number of prize tiers may never scale due to aggressive new algorithm](https://github.com/code-423n4/2023-08-pooltogether-mitigation-findings/issues/104) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/main/src/PrizePool.sol#L369
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/main/src/PrizePool.sol#L807-L811
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/main/src/abstract/TieredLiquidityDistributor.sol#L602-L619
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/main/src/abstract/TieredLiquidityDistributor.sol#L156
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/main/src/libraries/TierCalculationLib.sol#L134-L147


# Vulnerability details

## Comments
This issue is very similar to M-14 but covers another edge case where the threshold check is not performed when there are currently 14 prize tiers and at least 1 canary tier is claimed. This is due to an early return of `MAXIMUM_NUMBER_OF_TIERS`.

## Mitigation
The updated implementation has significantly changed the tier expansion logic so that it only depends on the total number of prize claims. The current number of tiers and the number of canary tier claims has no impact on the tier expansion logic and therefore the original issue has been resolved. However the updated logic has introduced a new issue.

Note: The same PR fixes multiple issues, but I have arbitrarily chosen to link this new issue with M-16.

## New issue
With the new tier expansion algorithm it is highly likely that the number of prize tiers will stagnate since the percentage of prizes that need to be claimed for a tier expansion increases as the number of tiers increase.

## Proof of Concept
When a draw is closed, the next number of tiers is computed based on the total number of claims with a call to `_computeNextNumberOfTiers`:

```
  function _computeNextNumberOfTiers(uint32 _claimCount) internal view returns (uint8) {
    // claimCount is expected to be the estimated number of claims for the current prize tier.
    uint8 numTiers = _estimateNumberOfTiersUsingPrizeCountPerDraw(_claimCount);
    return numTiers > MAXIMUM_NUMBER_OF_TIERS ? MAXIMUM_NUMBER_OF_TIERS : numTiers; // add new canary tier
  }
```

where the first part of the underlying `_estimateNumberOfTiersUsingPrizeCountPerDraw` method looks like:

```
  function _estimateNumberOfTiersUsingPrizeCountPerDraw(uint32 _prizeCount) internal view returns (uint8) {
    if (_prizeCount < ESTIMATED_PRIZES_PER_DRAW_FOR_4_TIERS) {
      return 3;
    } else if (_prizeCount < ESTIMATED_PRIZES_PER_DRAW_FOR_5_TIERS) {
      return 4;
```

Thus, for the number of tiers to increase from 3 to 4, the number of prize claims must be greater than or equal to `ESTIMATED_PRIZES_PER_DRAW_FOR_4_TIERS`:

```
ESTIMATED_PRIZES_PER_DRAW_FOR_4_TIERS = TierCalculationLib.estimatedClaimCount(3, _grandPrizePeriodDraws);
```

Here, the estimated claim count is the sum of the number of prizes for a tier multiplied by the tier odds for all the tiers:

```
  function estimatedClaimCount(
    uint8 _numberOfTiers,
    uint24 _grandPrizePeriod
  ) internal pure returns (uint32) {
    uint32 count = 0;
    for (uint8 i = 0; i < _numberOfTiers; i++) {
      count += uint32(
        uint256(
          unwrap(sd(int256(prizeCount(i))).mul(getTierOdds(i, _numberOfTiers, _grandPrizePeriod)))
        )
      );
    }
    return count;
  }
```

If we consider the actual number of prizes available for 3 tiers it is:
1 * 1/365 + 4 * 1 + 16 * 1 = 20 + 1/365 ~= 20

The return value of `ESTIMATED_PRIZES_PER_DRAW_FOR_4_TIERS` is:
1 * 1/365 + 4 * 1/sqrt(365) + 16 * 1 ~= 16

This difference is due to the fact that the `getTierOdds` in `TierCalculationLib.sol` doesn't return 1 for both the highest normal prize tier and the canary tier (it just returns 1 for the canary tier).

In this case, in order for the number of prize tiers to increase, at least 16 prizes (i.e. 80%) need to be claimed in the previous claim window. This doesn't sound too bad, but because the number of prizes per tier scales exponentially, the percentage of prizes that need to be claimed increases as the number of prize tiers increase.

The effect is that as a higher number of tiers are reached, a higher percentage of canary prizes need to be claimed in order to increase the number of prize tiers. The end result is that it is likely the number of prize tiers will stagnate despite there being enough liquidity available to increase the number of prize tiers.

## Tools Used
Manual review

## Recommendation
I would recommend applying a reverse exponential weighting to the number of prize claims required for a tier expansion based on the number of tiers. By this I mean that since the number of prizes per tier is 4^i, a similar weighting should be applied in the reverse direction to try to keep the percentage of prize claims required for a tier expansion relatively stable.

Alternatively, I would update `_estimateNumberOfTiersUsingPrizeCountPerDraw` to apply a fixed percentage to the actual number of prizes available (based on tiers odds of 1 for both the canary prize tier and the highest standard prize tier). I feel like this is the easier and also more logical fix.


## Assessed type

Math