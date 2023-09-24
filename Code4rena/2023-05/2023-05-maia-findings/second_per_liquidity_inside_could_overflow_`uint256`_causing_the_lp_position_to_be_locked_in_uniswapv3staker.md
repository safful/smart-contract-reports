## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-17

# [Second per liquidity inside could overflow `uint256` causing the LP position to be locked in UniswapV3Staker](https://github.com/code-423n4/2023-05-maia-findings/issues/505) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/uni-v3-staker/libraries/RewardMath.sol#L29


# Vulnerability details

## Impact

`UniswapV3Staker` depends on the second per liquidity inside values from the Uniswap V3 Pool to calculate the amount of reward a position should receive. This value represents the amount of second liquidity inside a tick range that is "active" (`tickLower < currentTick < tickUpper`). The second per liquidity inside a specific tick range is supposed to always increase over time.

In the `RewardMath` library, the seconds inside are calculated by taking the current timestamp value and subtracting the value at the moment the position is staked. Since this value increases over time, it should be normal. Additionally, this implementation is similar to [Uniswap Team's implementation](https://github.com/Uniswap/v3-staker/blob/4328b957701de8bed83689aa22c32eda7928d5ab/contracts/libraries/RewardMath.sol#L35).

```solidity
function computeBoostedSecondsInsideX128(
    uint256 stakedDuration,
    uint128 liquidity,
    uint128 boostAmount,
    uint128 boostTotalSupply,
    uint160 secondsPerLiquidityInsideInitialX128,
    uint160 secondsPerLiquidityInsideX128
) internal pure returns (uint160 boostedSecondsInsideX128) {
    // this operation is safe, as the difference cannot be greater than 1/stake.liquidity
    uint160 secondsInsideX128 = (secondsPerLiquidityInsideX128 - secondsPerLiquidityInsideInitialX128) * liquidity;
    // @audit secondPerLiquidityInsideX128 could smaller than secondsPerLiquidityInsideInitialX128
    ...
}
```

However, even though the second per liquidity inside value increases over time, it could overflow `uint256`, resulting in the calculation reverting. When `computeBoostedSecondsInsideX128()` reverts, function `_unstake()` will also revert, locking the LP position in the contract forever.


## Proof of Concept
Consider the value of the second per liquidity in three different timestamps: `t1 < t2 < t3`
```solidity
secondPerLiquidity_t1 = -10 = 2**256-10
secondPerLiquidity_t2 = 100
secondPerLiquidity_t3 = 300
```

As we can see, its value always increases over time, but the initial value could be smaller than 0. When calculating `computeBoostedSecondsInsideX128()` for a period from `t1 -> t2`, it will revert.

Additionally, as I mentioned earlier, this implementation is similar to the one from Uniswap team. However, please note that Uniswap team used Solidity 0.7, which won't revert on overflow and the formula works as expected while Maia uses Solidity 0.8.

For more information on how a tick is initialized, please refer to [this code](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Tick.sol#L132-L142)
```solidity
if (liquidityGrossBefore == 0) {
    // by convention, we assume that all growth before a tick was initialized happened _below_ the tick
    if (tick <= tickCurrent) {
        info.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
        info.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
        info.secondsPerLiquidityOutsideX128 = secondsPerLiquidityCumulativeX128;
        info.tickCumulativeOutside = tickCumulative;
        info.secondsOutside = time;
    }
    info.initialized = true;
}
```

The second per liquidity inside a range that has `tickLower < currentTick < tickUpper` is [calculated as](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L221):
```solidity
secondsPerLiquidityCumulativeX128 - tickLower.secondsPerLiquidityOutsideX128 - tickUpper.secondsPerLiquidityOutsideX128

// If lower tick is just init,
// Then: secondsPerLiquidityCumulativeX128 = tickLower.secondsPerLiquidityOutsideX128
// And: tickUpper.secondsPerLiquidityOutsideX128 != 0
// => Result will be overflow
```

## Tools Used

Manual Review

## Recommended Mitigation Steps
Consider using `unchecked` block to calculate this value.



## Assessed type

Under/Overflow