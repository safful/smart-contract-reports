## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-05

# [addDust does not achieve the goal correctly and may overflow revert](https://github.com/code-423n4/2023-08-goodentry-findings/issues/358) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/PositionManager/OptionsPositionManager.sol#L544-L551


# Vulnerability details

## Impact

The purpose of addDust is to ensure that both the token0Amount and token1Amount are greater than 100 units.
The current implementation is to calculate the value of 100 units 1e18 scales of token0 and token1, take the maximum value as liquidity, and add to the repayAmount.
The calculation does not take into account the actual getTokenAmounts result:
1. The calculated dust amount is based on the oracle price, while the actual amount consumed is based on the lp tick price
2. Even if the price of lp tick is equal to the spot price, which can't guarantee that the token0Amount and token1Amount of getTokenAmounts will be greater than 100 units.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../contracts/lib/LiquidityAmounts.sol";
import "../contracts/lib/TickMath.sol";

interface IERC20 {
    function decimals() external view returns (uint8);
}

interface IUniswapV3Pool{
  function slot0()
    external
    view
    returns (
      uint160 sqrtPriceX96,
      int24 tick,
      uint16 observationIndex,
      uint16 observationCardinality,
      uint16 observationCardinalityNext,
      uint8 feeProtocol,
      bool unlocked
    );
    
    function token0() external view returns (address);
    function token1() external view returns (address);
    function tickSpacing() external view returns (int24);
}

contract TestDust is Test {
    IUniswapV3Pool constant uniswapPool = IUniswapV3Pool(0xC2e9F25Be6257c210d7Adf0D4Cd6E3E881ba25f8);
    address constant token0 = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address constant token1 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    uint256 constant token0Price = 1e8;
    uint256 constant token1Price = 1830e8;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth", 17863266);
    }

    function testDustPrice() public {
        (uint token0Amount, uint token1Amount) = getTokenAmountsExcludingFees(getDust());
        console.log(token0Amount, token1Amount);
    }

    function getTokenAmountsExcludingFees(uint amount) public view returns (uint token0Amount, uint token1Amount){
        (uint160 sqrtPriceX96, int24 tick,,,,,)  = IUniswapV3Pool(uniswapPool).slot0();
        int24 tickSpacing  = IUniswapV3Pool(uniswapPool).tickSpacing();
        (token0Amount, token1Amount) = LiquidityAmounts.getAmountsForLiquidity(sqrtPriceX96, TickMath.getSqrtRatioAtTick(tick), TickMath.getSqrtRatioAtTick(tick + tickSpacing), uint128(amount));
    }

    function getDust() internal view returns (uint amount){
        uint scale0 = 10**(20 - IERC20(token0).decimals()) * token0Price / 1e8;
        uint scale1 = 10**(20 - IERC20(token1).decimals()) * token1Price / 1e8;

        if (scale0 > scale1) amount = scale0;
        else amount = scale1;
    }
}
```
For DAI/ETH pool, the token0Amount and token1Amount are 23278 and 0, dust value is close to 0, doesn't seem work.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../contracts/lib/LiquidityAmounts.sol";
import "../contracts/lib/TickMath.sol";

interface IERC20 {
    function decimals() external view returns (uint8);
}

interface IUniswapV3Pool{
  function slot0()
    external
    view
    returns (
      uint160 sqrtPriceX96,
      int24 tick,
      uint16 observationIndex,
      uint16 observationCardinality,
      uint16 observationCardinalityNext,
      uint8 feeProtocol,
      bool unlocked
    );
    
    function token0() external view returns (address);
    function token1() external view returns (address);
    function tickSpacing() external view returns (int24);
}

contract TestDust is Test {
    IUniswapV3Pool constant uniswapPool = IUniswapV3Pool(0xCBCdF9626bC03E24f779434178A73a0B4bad62eD);
    address constant token0 = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599;
    address constant token1 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    uint256 constant token0Price = 29000e8;
    uint256 constant token1Price = 1830e8;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth", 17863266);
    }

    function testDustPrice() public {
        (uint token0Amount, uint token1Amount) = getTokenAmountsExcludingFees(getDust());
    }

    function getTokenAmountsExcludingFees(uint amount) public view returns (uint token0Amount, uint token1Amount){
        (uint160 sqrtPriceX96, int24 tick,,,,,)  = IUniswapV3Pool(uniswapPool).slot0();
        int24 tickSpacing  = IUniswapV3Pool(uniswapPool).tickSpacing();
        (token0Amount, token1Amount) = LiquidityAmounts.getAmountsForLiquidity(sqrtPriceX96, TickMath.getSqrtRatioAtTick(tick), TickMath.getSqrtRatioAtTick(tick + tickSpacing), uint128(amount));
    }

    function getDust() internal view returns (uint amount){
        uint scale0 = 10**(20 - IERC20(token0).decimals()) * token0Price / 1e8;
        uint scale1 = 10**(20 - IERC20(token1).decimals()) * token1Price / 1e8;

        if (scale0 > scale1) amount = scale0;
        else amount = scale1;
    }
}
```
For BTC/ETH pool and current tick, the dust calculation will revert.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../contracts/lib/LiquidityAmounts.sol";
import "../contracts/lib/TickMath.sol";

interface IERC20 {
    function decimals() external view returns (uint8);
}

interface IUniswapV3Pool{
  function slot0()
    external
    view
    returns (
      uint160 sqrtPriceX96,
      int24 tick,
      uint16 observationIndex,
      uint16 observationCardinality,
      uint16 observationCardinalityNext,
      uint8 feeProtocol,
      bool unlocked
    );
    
    function token0() external view returns (address);
    function token1() external view returns (address);
    function tickSpacing() external view returns (int24);
}

contract TestDust is Test {
    IUniswapV3Pool constant uniswapPool = IUniswapV3Pool(0xCBCdF9626bC03E24f779434178A73a0B4bad62eD);
    address constant token0 = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599;
    address constant token1 = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    uint256 constant token0Price = 29000e8;
    uint256 constant token1Price = 1830e8;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth", 17863266);
    }

    function testDustPrice() public {
        (uint token0Amount, uint token1Amount) = getTokenAmountsExcludingFees(getDust());
    }

    function getTokenAmountsExcludingFees(uint amount) public view returns (uint token0Amount, uint token1Amount){
        (uint160 sqrtPriceX96, int24 tick,,,,,)  = IUniswapV3Pool(uniswapPool).slot0();
        int24 tickSpacing  = IUniswapV3Pool(uniswapPool).tickSpacing();
        (token0Amount, token1Amount) = LiquidityAmounts.getAmountsForLiquidity(sqrtPriceX96, TickMath.getSqrtRatioAtTick(tick - 10 * tickSpacing), TickMath.getSqrtRatioAtTick(tick), uint128(amount));
    }

    function getDust() internal view returns (uint amount){
        uint scale0 = 10**(20 - IERC20(token0).decimals()) * token0Price / 1e8;
        uint scale1 = 10**(20 - IERC20(token1).decimals()) * token1Price / 1e8;

        if (scale0 > scale1) amount = scale0;
        else amount = scale1;
    }
}
```

For the last 10 ticks, the token0Amount and token1Amount are 0 and 341236635433582778849. That's a huge number, not dust.

## Tools Used

Foundry

## Recommended Mitigation Steps

Check the amount of token0Amount and token1Amount corresponding to repayAmount instead of adding dust manually












## Assessed type

Context