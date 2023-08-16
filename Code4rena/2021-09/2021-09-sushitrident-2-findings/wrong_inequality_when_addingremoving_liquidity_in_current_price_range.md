## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Wrong inequality when adding/removing liquidity in current price range](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/34) 

# Handle

cmichel


# Vulnerability details

The `ConcentratedLiquidityPool.mint/burn` functions add/remove `liquidity` when `(priceLower < currentPrice && currentPrice < priceUpper)`.
Shouldn't it also be changed if `priceLower == currentPrice`?

## Impact
Pools that mint/burn liquidity at a time where the `currentPrice` is right at the lower price range do not work correctly and will lead to wrong swap amounts.

## Recommended Mitigation Steps
Change the inequalities to `if (priceLower <= currentPrice && currentPrice < priceUpper)`.


