## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-14

# [Chainlink's `latestRoundData` may return stale or incorrect result](https://github.com/code-423n4/2023-06-stader-findings/issues/15) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L646
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L648


# Vulnerability details

## Impact

Chainlink's `latestRoundData` is used here to retrieve price feed data, however there is insufficient protection against price staleness.

Return arguments other than `int256 answer` are necessary to determine the validity of the returned price, as it is possible for an outdated price to be received. See [here](https://ethereum.stackexchange.com/questions/133242/how-future-resilient-is-a-chainlink-price-feed/133843#133843) for reasons why a price feed might stop updating.

The return value `updatedAt` contains the timestamp at which the received price was last updated, and can be used to ensure that the price is not outdated. See more information about `latestRoundID` in the [Chainlink docs](https://docs.chain.link/data-feeds/api-reference#latestrounddata). Inaccurate price data can lead to functions not working as expected and/or lost funds.

## Proof of Concept

```solidity
    function getPORFeedData()
        internal
        view
        returns (
            uint256,
            uint256,
            uint256
        )
    {
        (, int256 totalETHBalanceInInt, , , ) = AggregatorV3Interface(staderConfig.getETHBalancePORFeedProxy())
            .latestRoundData();
        (, int256 totalETHXSupplyInInt, , , ) = AggregatorV3Interface(staderConfig.getETHXSupplyPORFeedProxy())
            .latestRoundData();
        return (uint256(totalETHBalanceInInt), uint256(totalETHXSupplyInInt), block.number);
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add a check for the `updatedAt` returned value from `latestRoundData`.

```diff
    function getPORFeedData()
        internal
        view
        returns (
            uint256,
            uint256,
            uint256
        )
    {
-       (, int256 totalETHBalanceInInt, , , ) = AggregatorV3Interface(staderConfig.getETHBalancePORFeedProxy())
+       (, int256 totalETHBalanceInInt, , uint256 balanceUpdatedAt, ) = AggregatorV3Interface(staderConfig.getETHBalancePORFeedProxy())
            .latestRoundData();
+       require(block.timestamp - balanceUpdatedAt <= MAX_DELAY, "stale price");

-       (, int256 totalETHXSupplyInInt, , , ) = AggregatorV3Interface(staderConfig.getETHXSupplyPORFeedProxy())
+       (, int256 totalETHXSupplyInInt, , uint256 supplyUpdatedAt, ) = AggregatorV3Interface(staderConfig.getETHXSupplyPORFeedProxy())
            .latestRoundData();
+       require(block.timestamp - supplyUpdatedAt <= MAX_DELAY, "stale price");

        return (uint256(totalETHBalanceInInt), uint256(totalETHXSupplyInInt), block.number);
    }
```


## Assessed type

Oracle