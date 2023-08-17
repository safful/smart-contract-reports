## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-20

# [Oracle does not treat upward and downward price movement the same in validity checks, causing safety issues in oracle usage.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/487) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L365


# Vulnerability details

## Description

NFTFloorOracle retrieves ERC721 prices for ParaSpace. maxPriceDeviation is a configurable parameter, which limits the change percentage from current price to a new feed update. We can see how priceDeviation is calculated and compared to maxPriceDeviation in \_checkValidity:

```
priceDeviation = _twap > _priorTwap
    ? (_twap * 100) / _priorTwap
    : (_priorTwap * 100) / _twap;
// config maxPriceDeviation as multiple directly(not percent) for simplicity
if (priceDeviation >= config.maxPriceDeviation) {
    return false;
}
return true;
```

The large number minus small number must be smaller than maxPriceDeviation. However, the way it is calculated means price decrease is much more sensitive and likely to be invalid than a price increase.

10 -> 15, priceDeviation = 15 / 10 = 1.5
15 -> 10, priceDeviation = 15 / 10 = 1.5

From 10 to 15, price rose by 50%. From 15 to 10, price  dropped by 33%. Both are the maximum change that would be allowed by deviation parameter. The effect of this behavior is that the protocol will be either too restrictive in how it accepts price drops, or too permissive in how it accepts price rises.

## Impact

Oracle does not treat upward and downward price movement the same in validity checks, causing safety issues in oracle usage.

## Proof of Concept

## Tools Used

Manual audit

## Recommended Mitigation Steps

Use a percentage base calculation for both upward and downward price movements.