## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-12

# [During oracle outages or feeder outages/disagreement, the `ParaSpaceFallbackOracle` is not used](https://github.com/code-423n4/2022-11-paraspace-findings/issues/420) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceOracle.sol#L114-L136


# Vulnerability details

## Impact
If the feeders haven't updated their prices within the default six-hour time window, the floor oracle will revert when requests are made to get the current price, and these reverts are not caught by the wrapper oracle which handles the fallback oracle, so rather than using the fallback oracle in these cases, the calls will revert, leading to liquidations to fail (see my other submission for the full chain from the floor oracle to the liquidation function).

Additionally, the code uses the deprecated chainlink `latestAnswer()` API for its token requests, which may revert once it is no longer supported

## Proof of Concept
`getPrice()` will fail if the values are stale:
```solidity
File: /paraspace-core/contracts/misc/NFTFloorOracle.sol   #1

236      function getPrice(address _asset)
237          external
238          view
239          override
240          returns (uint256 price)
241      {
242 @>       uint256 updatedAt = assetPriceMap[_asset].updatedAt;
243          require(
244              (block.number - updatedAt) <= config.expirationPeriod,
245 @>           "NFTOracle: asset price expired"
246          );
247          return assetPriceMap[_asset].twap;
248:     }
```
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L236-L248

They can be stale due to too much price skew, or the feeders being down, e.g. due to another bug:
```solidity
File: /paraspace-core/contracts/misc/NFTFloorOracle.sol   #2

369          // config maxPriceDeviation as multiple directly(not percent) for simplicity
370          if (priceDeviation >= config.maxPriceDeviation) {
371              return false;
372:         }
```
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L369-L372

```solidity
File: /paraspace-core/contracts/misc/NFTFloorOracle.sol   #3

376      function _finalizePrice(address _asset, uint256 _twap) internal {
377          PriceInformation storage assetPriceMapEntry = assetPriceMap[_asset];
378          assetPriceMapEntry.twap = _twap;
379 @>       assetPriceMapEntry.updatedAt = block.number;
380          assetPriceMapEntry.updatedTimestamp = block.timestamp;
381          emit AssetDataSet(
382              _asset,
383              assetPriceMapEntry.twap,
384              assetPriceMapEntry.updatedAt
385          );
386:     }
```
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L376-L386

The wrapper oracle does not use a try-catch, so it can't swallow these reverts and allow the fallback oracle to be used:
```solidity
File: /paraspace-core/contracts/misc/ParaSpaceOracle.sol   #4

114      /// @inheritdoc IPriceOracleGetter
115      function getAssetPrice(address asset)
116          public
117          view
118          override
119          returns (uint256)
120      {
121          if (asset == BASE_CURRENCY) {
122              return BASE_CURRENCY_UNIT;
123          }
124  
125          uint256 price = 0;
126          IEACAggregatorProxy source = IEACAggregatorProxy(assetsSources[asset]);
127          if (address(source) != address(0)) {
128 @>           price = uint256(source.latestAnswer());
129          }
130          if (price == 0 && address(_fallbackOracle) != address(0)) {
131 @>           price = _fallbackOracle.getAssetPrice(asset);
132          }
133  
134          require(price != 0, Errors.ORACLE_PRICE_NOT_READY);
135          return price;
136:     }
```
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceOracle.sol#L114-L136


## Tools Used
Code inspection

## Recommended Mitigation Steps
Use a try-catch when fetching the normal `latestAnswer()`, and use the fallback if it failed
