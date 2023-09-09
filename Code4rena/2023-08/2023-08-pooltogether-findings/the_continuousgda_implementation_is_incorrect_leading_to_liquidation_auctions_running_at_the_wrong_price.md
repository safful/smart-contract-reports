## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-10

# [The ContinuousGDA implementation is incorrect leading to liquidation auctions running at the wrong price](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/24) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L39-L41
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L65
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L86


# Vulnerability details

## Impact
The `LiquidationPair` contract facilitates Periodic Continuous Gradual Dutch Auctions for yield. This uses the underlying `ContinuousGDA.sol` library in order to correctly price the auctions.

However this library incorrectly implements the formula, using the emission rate in a few places where it should use the decay constant. Since the decay constant is usually less than the emission rate (as can also be seen from the test suite), this means that the `purchasePrice` calculation is lower than it should be, meaning that liquidations are over-incentivised.

## Proof of Concept
This is difficult to demonstrate given the issue is basically just that the formula in https://www.paradigm.xyz/2022/04/gda has been wrongly implemented. However I'll point out a few issues in the code:

```
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

In the `result` calculation you can see that `_k` is divided by `_emissionRate`. However, according to the proper formula, `_k` should be divided by `_decayConstant`.

Another issue occurs in `purchaseAmount` where `_k` is added to `lnParam` instead of ONE and `price` is multiplied by `_emissionRate` instead of `_decayConstant`:


```
  function purchaseAmount(
    SD59x18 _price,
    SD59x18 _emissionRate,
    SD59x18 _k,
    SD59x18 _decayConstant,
    SD59x18 _timeSinceLastAuctionStart
  ) internal pure returns (SD59x18) {
    if (_price.unwrap() == 0) {
      return SD59x18.wrap(0);
    }
    SD59x18 exp = _decayConstant.mul(_timeSinceLastAuctionStart).exp();
    SD59x18 lnParam = _k.add(_price.mul(_emissionRate).mul(exp)).div(_k);
    SD59x18 numerator = _emissionRate.mul(lnParam.ln());
    SD59x18 amount = numerator.div(_decayConstant);
    return amount;
  }
```

I would suggest double checking the formula and the derivation of the complementary formulas to calculate amount/k. The correct implementation is show in the diff below.

## Tools Used
Manual review

## Recommended Mitigation Steps
The implementation should be updated to correctly calculate the price for a continuous GDA. I have made the required fixes in the diff below:

```
diff --git a/src/libraries/ContinuousGDA.sol b/src/libraries/ContinuousGDA.sol
index 721d626..7e2bb61 100644
--- a/src/libraries/ContinuousGDA.sol
+++ b/src/libraries/ContinuousGDA.sol
@@ -36,9 +36,9 @@ library ContinuousGDA {
     bottomE = bottomE.exp();
     SD59x18 result;
     if (_emissionRate.unwrap() > 1e18) {
-      result = _k.div(_emissionRate).mul(topE).div(bottomE);
+      result = _k.div(_decayConstant).mul(topE).div(bottomE);
     } else {
-      result = _k.mul(topE.div(_emissionRate.mul(bottomE)));
+      result = _k.mul(topE.div(_decayConstant.mul(bottomE)));
     }
     return result;
   }
@@ -62,7 +62,7 @@ library ContinuousGDA {
       return SD59x18.wrap(0);
     }
     SD59x18 exp = _decayConstant.mul(_timeSinceLastAuctionStart).exp();
-    SD59x18 lnParam = _k.add(_price.mul(_emissionRate).mul(exp)).div(_k);
+    SD59x18 lnParam = ONE.add(_price.mul(_decayConstant).mul(exp)).div(_k);
     SD59x18 numerator = _emissionRate.mul(lnParam.ln());
     SD59x18 amount = numerator.div(_decayConstant);
     return amount;
@@ -83,7 +83,7 @@ library ContinuousGDA {
   ) internal pure returns (SD59x18) {
     SD59x18 exponent = _decayConstant.mul(_targetFirstSaleTime);
     SD59x18 eValue = exponent.exp();
-    SD59x18 multiplier = _emissionRate.mul(_price);
+    SD59x18 multiplier = _decayConstant.mul(_price);
     SD59x18 denominator = (_decayConstant.mul(_purchaseAmount).div(_emissionRate)).exp().sub(ONE);
     SD59x18 result = eValue.div(denominator);
     return result.mul(multiplier);

```





## Assessed type

Math