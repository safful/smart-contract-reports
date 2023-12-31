## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- LiquidityBasedTWAP

# [Oracle returns an improperly scaled USDV/VADER price](https://github.com/code-423n4/2021-12-vader-findings/issues/70) 

# Handle

TomFrenchBlockchain


# Vulnerability details

## Impact

Invalid values returned from oracle in vast majority of situations

## Proof of Concept

The LBT oracle does not properly scale values when calculating prices for VADER or USDV. To show this we consider the simplest case where we expect USDV to return a value of $1 and show that the oracle does not return this value.

Consider the case of the LBT oracle tracking a single USDV-DAI pair where USDV trades 1:1 for DAI and Chainlink reports that DAI is exactly $1. We then work through the lines linked below:

https://github.com/code-423n4/2021-12-vader/blob/00ed84015d4116da2f9db0c68db6742c89e73f65/contracts/lbt/LiquidityBasedTWAP.sol#L393-L409

For L397 we get a value of 1e8 as Chainlink reports the price of DAI with 8 decimals of accuracy.
```
foreignPrice = getChainlinkPrice(address(foreignAsset));
foreignPrice = 1e8
```

We can set `liquidityWeights[i]` and `totalUSDVLiquidityWeight` both to 1 as we only consider a single pair so L399-401 becomes
```
totalUSD = foreignPrice;
totalUSD = 1e8;
```

L403-408 is slightly more complex but from looking at the links below we can calculate `totalUSDV` as shown
https://github.com/code-423n4/2021-12-vader/blob/00ed84015d4116da2f9db0c68db6742c89e73f65/contracts/dex-v2/pool/VaderPoolV2.sol#L81-L90
https://github.com/code-423n4/2021-12-vader/blob/00ed84015d4116da2f9db0c68db6742c89e73f65/contracts/external/libraries/FixedPoint.sol#L137-L160

```
totalUSDV = pairData
    .nativeTokenPriceAverage
    .mul(pairData.foreignUnit)
    .decode144()
// pairData.nativeTokenPriceAverage == 2**112
// pairData.foreignUnit = 10**18
// decode144(x) = x >> 112
totalUSDV = (2**112).mul(10**18).decode144()
totalUSDV = 10**18
```

Using `totalUSD` and `totalUSDV` we can then calculate the return value of `_calculateUSDVPrice`

```
returnValue = (totalUSD * 1 ether) / totalUSDV;

returnValue = 1e8 * 1e18 / 1e18

returnValue = 1e8
```

For the oracle implementation to be correct we then expect that the Vader protocol to treat values of 1e8 from the oracle to mean USDV is worth $1. However from the lines of code linked below we can safely assume that it is intended to be that values of 1e18 represent $1 rather than 1e8.

https://github.com/code-423n4/2021-12-vader/blob/00ed84015d4116da2f9db0c68db6742c89e73f65/contracts/tokens/USDV.sol#L76
https://github.com/code-423n4/2021-12-vader/blob/00ed84015d4116da2f9db0c68db6742c89e73f65/contracts/tokens/USDV.sol#L109

High severity issue as the oracle is crucial for determining the exchange rate between VADER and USDV to be used for IL protection and minting/burning of USDV - an incorrect value will result in the protocol losing significant funds.

## Recommended Mitigation Steps

Go over oracle calculation again to ensure that various scale factors are properly accounted for. Some handling of the difference in the number of decimals between the chainlink oracle and the foreign asset should be added.

Build a test suite to ensure that the oracle returns the expected values for simple situations.

