## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-02

# [Use of slot0 to get sqrtPriceLimitX96 can lead to price manipulation.](https://github.com/code-423n4/2023-05-maia-findings/issues/823) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/cfed0dfa3bebdac0993b1b42239b4944eb0b196c/src/ulysses-omnichain/RootBridgeAgent.sol#L674
https://github.com/code-423n4/2023-05-maia/blob/cfed0dfa3bebdac0993b1b42239b4944eb0b196c/src/ulysses-omnichain/RootBridgeAgent.sol#L717


# Vulnerability details

## Impact
In the `RootBrigdeAgent.sol` the function's `_gasSwapOut` and `_gasSwapIn` uses `UniswapV3.slot0` to get the value of `sqrtPriceX96` which it use to perform the swap, however the `sqrtPriceX96` gotten from `Uniswap.slot0` is the most recent data point and can be manipulated easily via `MEV` bots & `Flashloans`⚡️ with sandwich attacks can cause lose of funds when interact with the `Uniswap.swap` function.

## Proof of Concept
You can see the `_gasSwapIn` function in the `RootBrigdeAgent.sol` [Link here](https://github.com/code-423n4/2023-05-maia/blob/cfed0dfa3bebdac0993b1b42239b4944eb0b196c/src/ulysses-omnichain/RootBridgeAgent.sol#L674C1-L689C75)

```solidity

     //Get sqrtPriceX96
        (uint160 sqrtPriceX96,,,,,,) = IUniswapV3Pool(poolAddress).slot0();

        // Calculate Price limit depending on pre-set price impact
        uint160 exactSqrtPriceImpact = (sqrtPriceX96 * (priceImpactPercentage / 2)) / GLOBAL_DIVISIONER;

        //Get limit
        uint160 sqrtPriceLimitX96 =
            zeroForOneOnInflow ? sqrtPriceX96 - exactSqrtPriceImpact : sqrtPriceX96 + exactSqrtPriceImpact;

        //Swap imbalanced token as long as we haven't used the entire amountSpecified and haven't reached the price limit
        try IUniswapV3Pool(poolAddress).swap(
            address(this),
            zeroForOneOnInflow,
            int256(_amount),
            sqrtPriceLimitX96,
            abi.encode(SwapCallbackData({tokenIn: gasTokenGlobalAddress}))
```
and You can see the `_gasSwapOut` function in the `RootBrigdeAgent.sol` [Link here](https://github.com/code-423n4/2023-05-maia/blob/cfed0dfa3bebdac0993b1b42239b4944eb0b196c/src/ulysses-omnichain/RootBridgeAgent.sol#L717C1-L734C11)

```solidity
   (uint160 sqrtPriceX96,,,,,,) = IUniswapV3Pool(poolAddress).slot0();

            // Calculate Price limit depending on pre-set price impact
            uint160 exactSqrtPriceImpact = (sqrtPriceX96 * (priceImpactPercentage / 2)) / GLOBAL_DIVISIONER;

            //Get limit
            sqrtPriceLimitX96 =
                zeroForOneOnInflow ? sqrtPriceX96 + exactSqrtPriceImpact : sqrtPriceX96 - exactSqrtPriceImpact;
        }

        //Swap imbalanced token as long as we haven't used the entire amountSpecified and haven't reached the price limit
        (int256 amount0, int256 amount1) = IUniswapV3Pool(poolAddress).swap(
            address(this),
            !zeroForOneOnInflow,
            int256(_amount),
            sqrtPriceLimitX96,
            abi.encode(SwapCallbackData({tokenIn: address(wrappedNativeToken)}))
        );
```

The both use the the `sqrtPriceX96` gotten from `Uniswap.slot0` 
An Attacker can Simply manipulate the `sqrtPriceX96` and if the `Uniswap.swap` function is called with the `sqrtPriceX96` the token will be bought at a higher price, and The Attacker would back run the transaction to sell thereby making gain but causing loss to whoever called those functions.

## Tools Used
`Manual Analysis`
## Recommended Mitigation Steps
Use The `TWAP` to get the value of `sqrtPriceX96`.


## Assessed type

MEV