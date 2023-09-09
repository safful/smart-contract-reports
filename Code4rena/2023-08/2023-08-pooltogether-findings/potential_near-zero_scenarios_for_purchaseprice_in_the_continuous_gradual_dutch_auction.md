## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-04

# [Potential Near-Zero Scenarios for purchasePrice in the Continuous Gradual Dutch Auction](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/122) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L211-L226
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L294-L319
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L16-L44


# Vulnerability details

## Impact
The Continuous Gradual Dutch Auction (CGDA) model has potential scenarios where the `purchasePrice` for an amount of tokens could approach near-zero values. This is influenced mainly by two factors: `_emissionRate` and `_timeSinceLastAuctionStart`. If either one or both of these factors (`_emissionRate` specifically more likely) are significantly large, the `purchasePrice` could drastically drop.

This condition could cause undesired economic effects in the auction process. Under this context, participants may acquire tokens (amount of Vault shares) at an extremely low price (very low POOL amount indeed), which could lead to significant chance of winnings.

## Proof of Concept
Here is the purchasePrice function within the ContinuousGDA library, which computes the purchase price of tokens based on various parameters.

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L23-L44

```solidity
  function purchasePrice(
    SD59x18 _amount,
    SD59x18 _emissionRate,
    SD59x18 _k,
    SD59x18 _decayConstant,
    SD59x18 _timeSinceLastAuctionStart
  ) internal pure returns (SD59x18) {
    if (_amount.unwrap() == 0) {
      return SD59x18.wrap(0);
    }
    SD59x18 topE = _decayConstant.mul(_amount).div(_emissionRate);
    topE = topE.exp().sub(ONE);
    SD59x18 bottomE = _decayConstant.mul(_timeSinceLastAuctionStart);
    bottomE = bottomE.exp();
    SD59x18 result;
    if (_emissionRate.unwrap() > 1e18) {
      result = _k.div(_emissionRate).mul(topE).div(bottomE);
    } else {
      result = _k.mul(topE.div(_emissionRate.mul(bottomE)));
    }
    return result;
  }
```
One possible scenarios where `purchasePrice` could approach near-zero values is:
_k = 1e18 (the initial price of the CGDA)
_decayConstant = 0.00001 (a very small decay constant)
_amount = 1e18 (the amount of tokens to purchase which has reportedly become a rare commodity)
_emissionRate = 1e20 (a significantly large, but more practical emission rate)
_timeSinceLastAuctionStart = 3600 (equivalent to 1 hour)

The small `_decayConstant`, along with the relatively short `_timeSinceLastAuctionStart`, results in `bottomE` being close to 1. `_emissionRate` is significantly larger than `_amount`, making `topE` also close to 1. As a result, this combination of factors drives the `purchasePrice` towards near-zero values. 

Please note that this is only one of many possible scenarios where the purchase price could approach near-zero values. Other combinations of parameter tweaks could potentially also lead to similar or more likely outcomes.

## Tools Used
Manual

## Recommended Mitigation Steps
1. Introduce checks in the `_computeExactAmountIn` function to ensure that `_emissionRate` doesn't exceed a certain practical limit.
2. Introduce checks in function `swapExactAmountOut` that the scaled ratio of `_amountInForPeriod` to `_amountOutForPeriod` should not fall below a certain threshold.






## Assessed type

Context