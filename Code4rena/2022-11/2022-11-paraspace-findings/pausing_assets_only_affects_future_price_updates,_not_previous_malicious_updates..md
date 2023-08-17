## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-21

# [Pausing assets only affects future price updates, not previous malicious updates.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/490) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L236


# Vulnerability details

## Description

NFTFloorOracle retrieves ERC721 prices for ParaSpace. It is pausable by admin on a per asset level using setPause(asset, flag).
setPrice will not be callable when asset is paused:
```
function setPrice(address _asset, uint256 _twap)
    public
    onlyRole(UPDATER_ROLE)
    onlyWhenAssetExisted(_asset)
    whenNotPaused(_asset)
```

However, getPrice() is unaffected by the pause flag. This is really dangerous behavior, because there will be 6 hours when the current price treated as valid, although the asset is clearly intended to be on lockdown. 

Basically, pauses are only forward facing, and whatever happened is valid. But, if we want to pause an asset, something fishy has already occured, or will occur by the time setPause() is called. So, "whatever happened happened" mentality is overly dangerous.

## Impact

Pausing assets only affects future price updates, not previous malicious updates.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Add whenNotPaused to getPrice() function as well.