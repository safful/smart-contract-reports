## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-12

# [Single hardcoded cap used for multiple tokens in a pump causing some assets to be more stale, while having no effects on other stable assets](https://github.com/code-423n4/2023-07-basin-findings/issues/55) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/pumps/MultiFlowPump.sol#L36-L37
https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/pumps/MultiFlowPump.sol#L205-L208
https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/pumps/MultiFlowPump.sol#L212-L215


# Vulnerability details

## Impact
When multiple tokens are handled in MultiFlowPump.sol, during trading, the pump might either unfairly reflects volatile assets to be more stale than normal trading activities, Or may fail to have any effects on relatively more stable assets. 

This could distort SMA and EMA token reserve values and unfairly reflect some token prices because it uses a one-size-fits-all approach to multiple token asset prices indiscriminate of their normal volatility levels.

## Proof of Concept
In MultiFlowPump.sol, a universal cap of maximum percentage change of a token reserve is set up as immutable variables - `LOG_MAX_INCREASE` and `LOG_MAX_DECREASE`. These two values are benchmarks for all tokens handled by the pump and are used to cap token reserve change per block in `_capReserve()`. And this cap is applied and checked on every single `update()` invoked by Well.sol.

```solidity
//MultiFlowPump.sol
    bytes16 immutable LOG_MAX_INCREASE;
    bytes16 immutable LOG_MAX_DECREASE;
```
```solidity
//MultiFlowPump.sol-update()
...
  for (uint256 i; i < numberOfReserves; ++i) {
            // Use a minimum of 1 for reserve. Geometric means will be set to 0 if a reserve is 0.
|>            pumpState.lastReserves[i] = _capReserve(
                pumpState.lastReserves[i], (reserves[i] > 0 ? reserves[i] : 1).fromUIntToLog2(), blocksPassed
            );
...
```
```solidity
//MultiFlowPump.sol-_capReserve()
...
        if (lastReserve.cmp(reserve) == 1) {
|>            bytes16 minReserve = lastReserve.add(blocksPassed.mul(LOG_MAX_DECREASE));
            // if reserve < minimum reserve, set reserve to minimum reserve
            if (minReserve.cmp(reserve) == 1) reserve = minReserve;
        }
        // Rerserve Increasing or staying the same.
        else {
|>            bytes16 maxReserve = blocksPassed.mul(LOG_MAX_INCREASE);
            maxReserve = lastReserve.add(maxReserve);
            // If reserve > maximum reserve, set reserve to maximum reserve
            if (reserve.cmp(maxReserve) == 1) reserve = maxReserve;
        }
        cappedReserve = reserve;
...
```
As seen from above, whenever a token reserve change is over the percentage dictated by `LOG_MAX_INCREASE` or `LOG_MAX_DECREASE`, the token reserve change will be capped at the maximum percentage and rewritten as calculated `minReserve` or `maxReserve` value. This is fine if the pump is only managing one trading pair(two tokens) because their trading volatility level can be determined. 

However, this is highly vulnerable when multiple trading pairs and multiple token assets are handled by a pump. And this is the intended scenario, which can be seen in Well.sol `_updatePumps()` where all token reserves handled by well will be passed to MultiFlowPump.sol. In this case, either volatile tokens will become more stale compared to their normal trading activities, or some stable tokens are allowed to deviate more than their normal trading activities which are vulnerable to exploits. 

See POC below as an example to show the above scenario. Full test file [here](https://gist.github.com/bzpassersby/bae15f0dec2ed656758ff468b734ec07). 
```solidity
//Pump.HardCap.t.sol
...
 function setUp() public {
        mWell = new MockReserveWell();
        initUser();
        pump = new MultiFlowPump(
            from18(0.5e18),
            from18(0.333333333333333333e18),
            12,
            from18(0.9e18)
        );
    }

    function test_HardCap() public {
        uint256[] memory initReserves = new uint256[](4);
        //initiate for mWell and pump
        initReserves[0] = 100e8; //wBTC
        initReserves[1] = 1600e18; //wETH
        initReserves[2] = 99000e4; //USDC
        initReserves[3] = 98000e4; //USDT
        mWell.update(address(pump), initReserves, new bytes(0));
        increaseTime(12);
        //Reserve update
        uint256[] memory updateReserves = new uint256[](4);
        updateReserves[0] = 160e8; //wBTC
        updateReserves[1] = 1000e18; //wETH
        updateReserves[2] = 96000e4; //USDC
        updateReserves[3] = 101000e4; //USDT
        mWell.update(address(pump), updateReserves, new bytes(0));
        // lastReserves0 reflects initReserves[i]
        uint256[] memory lastReserves0 = pump.readLastReserves(address(mWell));
        increaseTime(12);
        mWell.update(address(pump), updateReserves, new bytes(0));
        // lastReserves1 reflects 1st reserve update
        uint256[] memory lastReserves1 = pump.readLastReserves(address(mWell));
        assertEq(
            ((lastReserves1[0] - lastReserves0[0]) * 1000) / lastReserves0[0],
            500
        ); //wBTC: 50% reserve change versus 60% reserve change in well
        console.log(lastReserves1[0]); //14999999999
        assertEq(
            ((lastReserves0[1] - lastReserves1[1]) * 1000) / lastReserves0[1],
            333
        ); //wETH: 33% reserve change versus 37.5% reserve change in well
        console.log(lastReserves1[1]); //1066666666666666667199
        assertApproxEqAbs(lastReserves1[2], 96000e4, 1); //USDC: reserve change matches well, 3% change decrease
        assertApproxEqAbs(lastReserves1[3], 101000e4, 1); //USDT: reserve change matches well, 3% reserve increase
    }
...
```
As seen in POC, USDT/USDC reserve changes of more than 3% are allowed, whereas WBTC/WETH reserve changes are restricted. The test is based on 50% for max percentage per block increase and 33% max percentage per block decrease. Although the exact percentage change settings can be different and the actual token reserve changes per block differ by assets, the idea is such universal cap value likely will not fit multiple tokens or trading pairs. The cap can be either set too wide which is then no need to have a cap at all, or be set in a manner that restricts some volatile token assets.

See test results.
```
Running 1 test for test/pumps/Pump.HardCap.t.sol:PumpHardCapTest
[PASS] test_HardCap() (gas: 489346)
Logs:
  14999999999
  1066666666666666667199

Test result: ok. 1 passed; 0 failed; finished in 18.07ms

```

## Tools Used
Manual.
Vscode.
## Recommended Mitigation Steps
Different assets have different levels of volatility, and such variations in volatility are typically taken into account in an AMM or oracle. (Think UniswapV3 with different tick spacings, or chainlink with different price reporting intervals).

It’s better to avoid setting up `LOG_MAX_INCREASE`, `LOG_MIN_INCREASE` directly in MultiFlowPump.sol.
 
(1) Instead, in Well.sol allow `MAX_INCREASE` and `MAX_DECREASE` to be token specific and included as part of the immutable data created at the time of Well.sol development from Aquifer.sol, such that a token address can be included together with `MAX_INCREASE` and `MAX_DECREASE` as immutable constants.

(2) Then in `_updatePumps()`, pass current token reserves, and pass token `MAX_INCREASE` and `MAX_DECREASE` as `_pump.data` to MultiFlowPump.sol `update()`. 

(3) In MultiFlowPump.sol `update()` , when calculating `_capReserve()` , use decoded `_pump.data` to pass `MAX_INCREASE`, `MAX_DECREASE` for token specific cap calcuation.












## Assessed type

Oracle