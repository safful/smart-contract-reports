## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-22

# [Price can deviate by much more than maxDeviationRate](https://github.com/code-423n4/2022-11-paraspace-findings/issues/491) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L370


# Vulnerability details

## Description

NFTFloorOracle retrieves ERC721 prices for ParaSpace. maxPriceDeviation is a configurable parameter, which limits the change percentage from current price to a new feed update. 

```
function _checkValidity(address _asset, uint256 _twap)
    internal
    view
    returns (bool)
{
    require(_twap > 0, "NFTOracle: price should be more than 0");
    PriceInformation memory assetPriceMapEntry = assetPriceMap[_asset];
    uint256 _priorTwap = assetPriceMapEntry.twap;
    uint256 _updatedAt = assetPriceMapEntry.updatedAt;
    uint256 priceDeviation;
    //first price is always valid
    if (_priorTwap == 0 || _updatedAt == 0) {
        return true;
    }
    priceDeviation = _twap > _priorTwap
        ? (_twap * 100) / _priorTwap
        : (_priorTwap * 100) / _twap;
    // config maxPriceDeviation as multiple directly(not percent) for simplicity
    if (priceDeviation >= config.maxPriceDeviation) {
        return false;
    }
    return true;
}
```

The issue is that it only mitigates a massive change in a single transaction, but attackers can still just call setPrice() and update the price many times in a row, each with maxDevicePrice in price change. 

The correct approach would be to limit the TWAP growth or decline over some timespan. This will allow admins to react in time to a potential attack and remove the bad price feed.

## Impact

Price can deviate by much more than maxDeviationRate

## Proof of Concept

maxDeviationRate = 150
Currently 3 oracles in place, their readings are [1, 1.1, 2]
Oracle #2 can call setPrice(1.5), setPrice(1.7), setPrice(2) to immediately change the price from 1.1 to 2, which is almost 100% growth, much above 50% allowed growth.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Code already has lastUpdated timestamp for each oracle. Use the elapsed time to calculate a reasonable maximum price change per oracle.