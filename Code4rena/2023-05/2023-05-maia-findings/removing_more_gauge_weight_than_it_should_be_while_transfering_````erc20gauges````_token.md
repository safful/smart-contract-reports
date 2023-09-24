## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-21

# [Removing more gauge weight than it should be while transfering ````ERC20Gauges```` token](https://github.com/code-423n4/2023-05-maia-findings/issues/477) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Gauges.sol#L536


# Vulnerability details

## Impact
The ````_decrementWeightUntilFree()```` function is not well implemented. If there are deprecated gauges, it would remove more gauge weight than it should be while transfering ````ERC20Gauges```` token.

## Proof of Concept
The issue arises on L536 of ````_decrementWeightUntilFree()````, where ````userFreed````, rather than ````totalFreed````, should be used in loop condition.
```diff
File: src\erc-20\ERC20Gauges.sol
519:     function _decrementWeightUntilFree(address user, uint256 weight) internal nonReentrant {
520:         uint256 userFreeWeight = freeVotes(user) + userUnusedVotes(user);
521: 
522:         // early return if already free
523:         if (userFreeWeight >= weight) return;
524: 
525:         uint32 currentCycle = _getGaugeCycleEnd();
526: 
527:         // cache totals for batch updates
528:         uint112 userFreed;
529:         uint112 totalFreed;
530: 
531:         // Loop through all user gauges, live and deprecated
532:         address[] memory gaugeList = _userGauges[user].values();
533: 
534:         // Free gauges through the entire list or until underweight
535:         uint256 size = gaugeList.length;
-536:         for (uint256 i = 0; i < size && (userFreeWeight + totalFreed) < weight;) {
+536:         for (uint256 i = 0; i < size && (userFreeWeight + userFreed) < weight;) {
537:             address gauge = gaugeList[i];
538:             uint112 userGaugeWeight = getUserGaugeWeight[user][gauge];
539:             if (userGaugeWeight != 0) {
540:                 // If the gauge is live (not deprecated), include its weight in the total to remove
541:                 if (!_deprecatedGauges.contains(gauge)) {
542:                     totalFreed += userGaugeWeight;
543:                 }
544:                 userFreed += userGaugeWeight;
545:                 _decrementGaugeWeight(user, gauge, userGaugeWeight, currentCycle);
546: 
547:                 unchecked {
548:                     i++;
549:                 }
550:             }
551:         }
552: 
553:         getUserWeight[user] -= userFreed;
554:         _writeGaugeWeight(_totalWeight, _subtract112, totalFreed, currentCycle);
555:     }
556: }
```
The following test script shows how excess gauge weight is inadvertently removed during the transfer of ````ERC20Gauges```` tokens.
````FilePath: test\erc-20\ERC20GaugesBug.t.sol````
```solidity
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import {console2} from "forge-std/console2.sol";

import {DSTestPlus} from "solmate/test/utils/DSTestPlus.sol";

import {MockBaseV2Gauge, FlywheelGaugeRewards, ERC20} from "../gauges/mocks/MockBaseV2Gauge.sol";

import {MockERC20Gauges, ERC20Gauges} from "./mocks/MockERC20Gauges.t.sol";

contract ERC20GaugesTest is DSTestPlus {
    MockERC20Gauges token;
    address gauge1;
    address gauge2;

    function setUp() public {
        token = new MockERC20Gauges(address(this), 3600, 600); // 1 hour cycles, 10 minute freeze

        hevm.mockCall(address(0), abi.encodeWithSignature("rewardToken()"), abi.encode(ERC20(address(0xDEAD))));
        hevm.mockCall(address(0), abi.encodeWithSignature("gaugeToken()"), abi.encode(ERC20Gauges(address(0xBEEF))));
        hevm.mockCall(
            address(this), abi.encodeWithSignature("bHermesBoostToken()"), abi.encode(ERC20Gauges(address(0xBABE)))
        );

        gauge1 = address(new MockBaseV2Gauge(FlywheelGaugeRewards(address(0)), address(0), address(0)));
        gauge2 = address(new MockBaseV2Gauge(FlywheelGaugeRewards(address(0)), address(0), address(0)));
    }

    function testRemovingMoreGaugeWeightThanItShouldBe() public {
        // initializing
        token.setMaxGauges(2);
        token.addGauge(gauge1);
        token.addGauge(gauge2);
        token.setMaxDelegates(2);

        // test users
        address alice = address(0x111);
        address bob = address(0x222);

        // give some token to alice 
        token.mint(alice, 200);

        // alice delegate votes to self
        hevm.prank(alice);
        token.delegate(alice);
        assertEq(token.getVotes(alice), 200);

        // alice increments gauge1 and gauge2 with weight 100 respectively
        hevm.startPrank(alice);
        token.incrementGauge(gauge1, 100);
        token.incrementGauge(gauge2, 100);
        hevm.stopPrank();
        assertEq(token.getUserGaugeWeight(alice, gauge1), 100);
        assertEq(token.getUserGaugeWeight(alice, gauge2), 100);
        assertEq(token.getUserWeight(alice), 200);

        // removing gauge1 would trigger the bug
        token.removeGauge(gauge1);

        // transfer only 100 weight
        hevm.prank(alice);
        token.transfer(bob, 100);

        // but all 200 weight is removed, and the 100 weight of gauge2
        // is removed unnecessarily
        assertEq(token.getUserGaugeWeight(alice, gauge1), 0);
        assertEq(token.getUserGaugeWeight(alice, gauge2), 0);
        assertEq(token.getUserWeight(alice), 0);
    }

}
```

Test log:
```solidity
2023-05-maia> forge test --match-test testRemovingMoreGaugeWeightThanExpected -vv
[⠘] Compiling...
[⠆] Compiling 87 files with 0.8.18
[⠘] Solc 0.8.18 finished in 39.43s
Compiler run successful

Running 1 test for test/erc-20/ERC20GaugesBug.t.sol:ERC20GaugesTest
[PASS] testRemovingMoreGaugeWeightThanExpected() (gas: 649072)
Test result: ok. 1 passed; 0 failed; finished in 2.64ms
```

## Tools Used
Manually review

## Recommended Mitigation Steps
See PoC


## Assessed type

Other