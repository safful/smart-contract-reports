## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-15

# [NFTFloorOracle's assets will use old prices if added back after removal](https://github.com/code-423n4/2022-11-paraspace-findings/issues/459) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L296-L305


# Vulnerability details

`assetFeederMap` mapping has elements of the `FeederRegistrar` structure type, that contains nested `feederPrice` mapping. When an asset is being removed with _removeAsset(), its `assetFeederMap` entry is deleted with the plain `delete assetFeederMap[_asset]` operation, which leaves `feederPrice` mapping intact.

This way if the asset be added back after removal its old prices will be reused via _combine() given that their timestamps are fresh enough and the corresponding feeders stay active.

## Impact

Old prices can be irrelevant in the big enough share of cases. The asset can be removed due to its internal issues that are usually coupled with price disruptions, so it is reasonable to assume that recent prices of a removed asset can be corrupted for the same reason that caused its removal.

Nevertheless these prices will be used as if they were added for this asset after its return. Recency logic will work as usual, so the issue is conditional on `config.expirationPeriod` being substantial enough, which might be the case for all illiquid assets.

Net impact is incorrect valuation of the corresponding NFTs, that can lead to the liquidation of the healthy accounts, which is the permanent loss of the principal funds for their owners. However, due to prerequisites placing the severity to be medium.

## Proof of Concept

_removeAsset() will `delete` the `assetFeederMap` entry, setting its elements to zero:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L296-L305

```solidity
    function _removeAsset(address _asset)
        internal
        onlyWhenAssetExisted(_asset)
    {
        uint8 assetIndex = assetFeederMap[_asset].index;
        delete assets[assetIndex];
        delete assetPriceMap[_asset];
        delete assetFeederMap[_asset];
        emit AssetRemoved(_asset);
    }
```

Notice, that `assetFeederMap` is the mapping with `FeederRegistrar` elements:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L86-L88

```solidity
    /// @dev Original raw value to aggregate with
    // the NFT contract address -> FeederRegistrar which contains price from each feeder
    mapping(address => FeederRegistrar) public assetFeederMap;
```

`FeederRegistrar` is a structure with a nested `feederPrice` mapping.

`feederPrice` nested mapping will not be deleted with `delete assetFeederMap[_asset]`:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L32-L41

```solidity
struct FeederRegistrar {
    // if asset registered or not
    bool registered;
    // index in asset list
    uint8 index;
    // if asset paused,reject the price
    bool paused;
    // feeder -> PriceInformation
    mapping(address => PriceInformation) feederPrice;
}
```

Per operation docs `So if you delete a struct, it will reset all members that are not mappings and also recurse into the members unless they are mappings`:

https://docs.soliditylang.org/en/latest/types.html#delete

This way, if the asset be added back its `feederPrice` mapping will be reused.

On addition of this old `_asset` only its `assetFeederMap[_asset].index` be renewed:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L278-L286

```solidity
    function _addAsset(address _asset)
        internal
        onlyWhenAssetNotExisted(_asset)
    {
        assetFeederMap[_asset].registered = true;
        assets.push(_asset);
        assetFeederMap[_asset].index = uint8(assets.length - 1);
        emit AssetAdded(_asset);
    }
```

This means that the old prices will be immediately used for price construction, given that `config.expirationPeriod` is big enough:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L397-L430

```solidity
    function _combine(address _asset, uint256 _twap)
        internal
        view
        returns (bool, uint256)
    {
        FeederRegistrar storage feederRegistrar = assetFeederMap[_asset];
        uint256 currentBlock = block.number;
        //first time just use the feeding value
        if (assetPriceMap[_asset].twap == 0) {
            return (true, _twap);
        }
        //use memory here so allocate with maximum length
        uint256 feederSize = feeders.length;
        uint256[] memory validPriceList = new uint256[](feederSize);
        uint256 validNum = 0;
        //aggeregate with price from all feeders
        for (uint256 i = 0; i < feederSize; i++) {
            PriceInformation memory priceInfo = feederRegistrar.feederPrice[
                feeders[i]
            ];
            if (priceInfo.updatedAt > 0) {
                uint256 diffBlock = currentBlock - priceInfo.updatedAt;
                if (diffBlock <= config.expirationPeriod) {
                    validPriceList[validNum] = priceInfo.twap;
                    validNum++;
                }
            }
        }
        if (validNum < MIN_ORACLES_NUM) {
            return (false, assetPriceMap[_asset].twap);
        }
        _quickSort(validPriceList, 0, int256(validNum - 1));
        return (true, validPriceList[validNum / 2]);
    }
```

## Recommended Mitigation Steps

Consider clearing the current list of prices on asset removal, for example:

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L296-L305

```solidity
    function _removeAsset(address _asset)
        internal
        onlyWhenAssetExisted(_asset)
    {
+       FeederRegistrar storage feederRegistrar = assetFeederMap[_asset];
+       uint256 feederSize = feeders.length;
+       for (uint256 i = 0; i < feederSize; i++) {
+           delete feederRegistrar.feederPrice[feeders[i]];
+       }
        uint8 assetIndex = feederRegistrar.index;
        delete assets[assetIndex];
        delete assetPriceMap[_asset];
        delete assetFeederMap[_asset];
        emit AssetRemoved(_asset);
    }
```