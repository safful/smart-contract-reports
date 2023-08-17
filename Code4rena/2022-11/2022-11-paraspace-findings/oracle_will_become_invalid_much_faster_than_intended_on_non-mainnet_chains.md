## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-23

# [Oracle will become invalid much faster than intended on non-mainnet chains](https://github.com/code-423n4/2022-11-paraspace-findings/issues/496) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/NFTFloorOracle.sol#L12


# Vulnerability details

## Description

NFTFloorOracle is in charge of answering price queries for ERC721 assets. 
EXPIRATION_PERIOD constant is the max amount of blocks allowed to have passed for the reading to be considered up to date:
```
uint256 diffBlock = currentBlock - priceInfo.updatedAt;
if (diffBlock <= config.expirationPeriod) {
    validPriceList[validNum] = priceInfo.twap;
    validNum++;
}
```

We can see it is set to 1800, which is intended to be 6 hours with 12 second block time assumption:
```
//expirationPeriod at least the interval of client to feed data(currently 6h=21600s/12=1800 in mainnet)
//we do not accept price lags behind to much
uint128 constant EXPIRATION_PERIOD = 1800;
```

The issue is that different blockchains have wildly different block times. BSC has one every 3 seconds, while Avalanche has a new block every 1 sec. Also the block product rate may be subject to future changes. Therefore, readings may be considered stale much faster than intended on non-mainnet chains. 

The correct EVM compatible way to check it is using the block.timestamp variable. Make sure the difference between current and previous timestamp is under 6 * 3600 seconds.

Paraspace docs show they are clearly intending to deploy in multiple chains so it's very relevant.

## Impact

Oracle will become invalid much faster than intended on non-mainnet chains

## Tools Used

Manual audit

## Recommended Mitigation Steps

Use block.timestamp to measure passage of time.