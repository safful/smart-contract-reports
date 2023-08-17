## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- H-05

# [Attacker can manipulate low TVL Uniswap V3 pool to borrow and swap to make Lending Pool in loss.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/407) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/UniswapV3OracleWrapper.sol#L176


# Vulnerability details

## Impact

In Paraspace protocol, any Uniswap V3 position that are consist of ERC20 tokens that Paraspace support can be used as collateral to borrow funds from Paraspace pool. The value of the Uniswap V3 position will be sum of value of ERC20 tokens in it.

```solidity
function getTokenPrice(uint256 tokenId) public view returns (uint256) {
    UinswapV3PositionData memory positionData = getOnchainPositionData(
        tokenId
    );

    PairOracleData memory oracleData = _getOracleData(positionData);

    (uint256 liquidityAmount0, uint256 liquidityAmount1) = LiquidityAmounts
        .getAmountsForLiquidity(
            oracleData.sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(positionData.tickLower),
            TickMath.getSqrtRatioAtTick(positionData.tickUpper),
            positionData.liquidity
        );

    (
        uint256 feeAmount0,
        uint256 feeAmount1
    ) = getLpFeeAmountFromPositionData(positionData);

    return // @audit can be easily manipulated with low TVL pool
        (((liquidityAmount0 + feeAmount0) * oracleData.token0Price) /
            10**oracleData.token0Decimal) +
        (((liquidityAmount1 + feeAmount1) * oracleData.token1Price) /
            10**oracleData.token1Decimal);
}
```


However, Uniswap V3 can have multiple pools for the **same pairs** of ERC20 tokens with **different fee** params. A fews has most the liquidity, while other pools have extremely little TVL or even not created yet. Attackers can abuse it, create low TVL pool where liquidity in this pool mostly (or fully) belong to attacker’s position, deposit this position as collateral and borrow token in Paraspace pool, swap to make the original position reduce the original value and cause Paraspace pool in loss.

## Proof of Concept

Consider the scenario where WETH and DAI are supported as collateral in Paraspace protocol. 
1. Alice (attacker) create a new WETH/DAI pool in Uniswap V3 and add liquidity with the following amount
`1e18 wei WETH - 1e6 wei DAI = 1 WETH - 1e-12 DAI ~= 1 ETH`
Let's just assume Alice position has price range from [MIN_TICK, MAX_TICK] so the math can be approximately like Uniswap V2 - constant product. 
Note that this pool only has liquidity from Alice. 
2. Alice deposit this position into Paraspace, value of this position is approximately `1 WETH` and Alice borrow maximum possible amount of USDC.
3. Alice make swap in her WETH/DAI pool in Uniswap V3 to make the position become
`1e6 wei WETH - 1e18 wei DAI = 1e-12 WETH - 1 DAI ~= 1 DAI`

Please note that the math I've done above is approximation based on Uniswap V2 formula `x * y = k` because Alice provided liquidity from MIN_TICK to MAX_TICK. 
For more information about Uniswap V3 formula, please check their whitepaper here: https://uniswap.org/whitepaper-v3.pdf


## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider adding whitelist, only allowing pool with enough TVL to be collateral in Paraspace protocol.
