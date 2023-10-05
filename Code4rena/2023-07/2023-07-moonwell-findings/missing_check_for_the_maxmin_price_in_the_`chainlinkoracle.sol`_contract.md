## Tags

- bug
- 2 (Med Risk)
- low quality report
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-02

# [missing check for the max/min price in the `chainlinkOracle.sol` contract](https://github.com/code-423n4/2023-07-moonwell-findings/issues/340) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Oracles/ChainlinkOracle.sol#L97-L113


# Vulnerability details

## Impact

the `chainlinkOracle.sol` contract specially the `getChainlinkPrice` function using the aggregator v2 and v3 to get/call the `latestRoundData`. the function should check for the min and max amount return to prevent some case happen, something like this:

https://solodit.xyz/issues/missing-checks-for-chainlink-oracle-spearbit-connext-pdf
https://solodit.xyz/issues/m-16-chainlinkadapteroracle-will-return-the-wrong-price-for-asset-if-underlying-aggregator-hits-minanswer-sherlock-blueberry-blueberry-git

if case like luna happen then the oracle will return the minimum price and not the crashed price.

## Proof of Concept

the function `getChainlinkPrice`:

```soliditiy
function getChainlinkPrice(
        AggregatorV3Interface feed
    ) internal view returns (uint256) {
        (, int256 answer, , uint256 updatedAt, ) = AggregatorV3Interface(feed)
            .latestRoundData();
        require(answer > 0, "Chainlink price cannot be lower than 0");
        require(updatedAt != 0, "Round is in incompleted state");

        // Chainlink USD-denominated feeds store answers at 8 decimals
        uint256 decimalDelta = uint256(18).sub(feed.decimals());
        // Ensure that we don't multiply the result by 0
        if (decimalDelta > 0) {
            return uint256(answer).mul(10 ** decimalDelta);
        } else {
            return uint256(answer);
        }
    }
```

the function did not check for the min and max price.

## Tools Used

manual review

## Recommended Mitigation Steps

some check like this can be added to avoid returning of the min price or the max price in case of the price crashes.

```solidity
          require(answer < _maxPrice, "Upper price bound breached");
          require(answer > _minPrice, "Lower price bound breached");
```





## Assessed type

Oracle