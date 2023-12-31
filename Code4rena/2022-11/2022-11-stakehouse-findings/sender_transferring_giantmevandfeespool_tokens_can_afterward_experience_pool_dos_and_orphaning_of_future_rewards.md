## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-12

# [Sender transferring GiantMevAndFeesPool tokens can afterward experience pool DOS and orphaning of future rewards](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/178) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/GiantMevAndFeesPool.sol#L170-L173


# Vulnerability details

## Impact
When a user transfers away GiantMevAndFeesPool tokens, the pool's claimed[] computed is left unchanged and still corresponds to what they had claimed with their old (higher) number of tokens. (See GiantMevAndFeesPool afterTokenTransfer() - no adjustment is made to claimed[] on the from side.) As a result, their claimed[] may be higher than the max amount they could possibly have claimed for their new (smaller) number of tokens. The erroneous claimed value can cause an integer overflow when the claimed[] value is subtracted, leading to inability for this user to access some functions of the GiantMevAndFeesPool - including such things as being able to transfer their tokens (overflow is triggered in a callback attempting to pay out their rewards). These overflows will occur in SyndicateRewardsProcessor's _previewAccumulatedETH() and _distributeETHRewardsToUserForToken(), the latter of which is called in a number of places. When rewards are later accumulated in the pool, the user will not be able to claim certain rewards owed to them because of the incorrect (high) claimed[] value. The excess rewards will be orphaned in the pool.

## Proof of Concept
This patch demonstrates both DOS and orphaned rewards due to the claimed[] error described above. Note that the patch includes a temp fix for the separate issue calculating claimed[] in _distributeETHRewardsToUserForToken() in order to demonstrate this is a separate issue.

Run test
```
forge test -m testTransferDOSUserOrphansFutureRewards
```

Patch
```diff
diff --git a/contracts/liquid-staking/SyndicateRewardsProcessor.sol b/contracts/liquid-staking/SyndicateRewardsProcessor.sol
index 81be706..ca44ae6 100644
--- a/contracts/liquid-staking/SyndicateRewardsProcessor.sol
+++ b/contracts/liquid-staking/SyndicateRewardsProcessor.sol
@@ -60,7 +60,7 @@ abstract contract SyndicateRewardsProcessor {
             // Calculate how much ETH rewards the address is owed / due 
             uint256 due = ((accumulatedETHPerLPShare * balance) / PRECISION) - claimed[_user][_token];
             if (due > 0) {
-                claimed[_user][_token] = due;
+                claimed[_user][_token] += due; // temp fix claimed calculation
 
                 totalClaimed += due;
 
diff --git a/test/foundry/GiantPools.t.sol b/test/foundry/GiantPools.t.sol
index 7e8bfdb..6468373 100644
--- a/test/foundry/GiantPools.t.sol
+++ b/test/foundry/GiantPools.t.sol
@@ -5,14 +5,18 @@ pragma solidity ^0.8.13;
 import "forge-std/console.sol";
 import { TestUtils } from "../utils/TestUtils.sol";
 
+import { MockLiquidStakingManager } from "../../contracts/testing/liquid-staking/MockLiquidStakingManager.sol";
 import { GiantSavETHVaultPool } from "../../contracts/liquid-staking/GiantSavETHVaultPool.sol";
 import { GiantMevAndFeesPool } from "../../contracts/liquid-staking/GiantMevAndFeesPool.sol";
 import { LPToken } from "../../contracts/liquid-staking/LPToken.sol";
+import { GiantLP } from "../../contracts/liquid-staking/GiantLP.sol";
 import { MockSlotRegistry } from "../../contracts/testing/stakehouse/MockSlotRegistry.sol";
 import { MockSavETHVault } from "../../contracts/testing/liquid-staking/MockSavETHVault.sol";
 import { MockGiantSavETHVaultPool } from "../../contracts/testing/liquid-staking/MockGiantSavETHVaultPool.sol";
 import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 
+import "forge-std/console.sol";
+
 contract GiantPoolTests is TestUtils {
 
     MockGiantSavETHVaultPool public giantSavETHPool;
@@ -116,4 +120,171 @@ contract GiantPoolTests is TestUtils {
         assertEq(dETHToken.balanceOf(savETHUser), 24 ether);
     }
 
+    function addNewLSM(address payable giantFeesAndMevPool, bytes memory blsPubKey) public returns (address payable) {
+        manager = deployNewLiquidStakingNetwork(
+            factory,
+            admin,
+            true,
+            "LSDN"
+        );
+
+        savETHVault = MockSavETHVault(address(manager.savETHVault()));
+
+        giantSavETHPool = new MockGiantSavETHVaultPool(factory, savETHVault.dETHToken());
+
+        // Set up users and ETH
+        address nodeRunner = accountOne; vm.deal(nodeRunner, 12 ether);
+        address savETHUser = accountThree; vm.deal(savETHUser, 24 ether);
+
+        // Register BLS key
+        registerSingleBLSPubKey(nodeRunner, blsPubKey, accountFour);
+
+        // Deposit ETH into giant savETH
+        vm.prank(savETHUser);
+        giantSavETHPool.depositETH{value: 24 ether}(24 ether);
+        assertEq(giantSavETHPool.lpTokenETH().balanceOf(savETHUser), 24 ether);
+        assertEq(address(giantSavETHPool).balance, 24 ether);
+
+        // Deploy ETH from giant LP into savETH pool of LSDN instance
+        bytes[][] memory blsKeysForVaults = new bytes[][](1);
+        blsKeysForVaults[0] = getBytesArrayFromBytes(blsPubKey);
+
+        uint256[][] memory stakeAmountsForVaults = new uint256[][](1);
+        stakeAmountsForVaults[0] = getUint256ArrayFromValues(24 ether);
+
+        giantSavETHPool.batchDepositETHForStaking(
+            getAddressArrayFromValues(address(manager.savETHVault())),
+            getUint256ArrayFromValues(24 ether),
+            blsKeysForVaults,
+            stakeAmountsForVaults
+        );
+        assertEq(address(manager.savETHVault()).balance, 24 ether);
+
+        assert(giantFeesAndMevPool.balance >= 4 ether);
+        stakeAmountsForVaults[0] = getUint256ArrayFromValues(4 ether);
+        GiantMevAndFeesPool(giantFeesAndMevPool).batchDepositETHForStaking(
+            getAddressArrayFromValues(address(manager.stakingFundsVault())),
+            getUint256ArrayFromValues(4 ether),
+            blsKeysForVaults,
+            stakeAmountsForVaults
+        );
+
+        // Ensure we can stake and mint derivatives
+        stakeAndMintDerivativesSingleKey(blsPubKey);
+
+        return payable(manager);
+    }
+
+    function testTransferDOSUserOrphansFutureRewards() public {
+
+        address feesAndMevUserOne = accountTwo; vm.deal(feesAndMevUserOne, 8 ether);
+        address feesAndMevUserTwo = accountFour;
+
+       // Deposit ETH into giant fees and mev
+        vm.startPrank(feesAndMevUserOne);
+        giantFeesAndMevPool.depositETH{value: 8 ether}(8 ether);
+        vm.stopPrank();
+
+        MockLiquidStakingManager manager1 = MockLiquidStakingManager(addNewLSM(payable(giantFeesAndMevPool), blsPubKeyOne));
+        MockLiquidStakingManager manager2 = MockLiquidStakingManager(addNewLSM(payable(giantFeesAndMevPool), blsPubKeyTwo));
+
+        bytes[][] memory blsPubKeyOneInput = new bytes[][](1);
+        blsPubKeyOneInput[0] = getBytesArrayFromBytes(blsPubKeyOne);
+
+        bytes[][] memory blsPubKeyTwoInput = new bytes[][](1);
+        blsPubKeyTwoInput[0] = getBytesArrayFromBytes(blsPubKeyTwo);
+
+        vm.warp(block.timestamp + 3 hours);
+
+        // Add 2 eth rewards to manager1's staking funds vault.
+        vm.deal(address(manager1.stakingFundsVault()), 2 ether);
+
+        // Claim rewards into the giant pool and distribute them to user one.
+        vm.startPrank(feesAndMevUserOne);
+        giantFeesAndMevPool.claimRewards(
+            feesAndMevUserOne,
+            getAddressArrayFromValues(address(manager1.stakingFundsVault())),
+            blsPubKeyOneInput);
+        vm.stopPrank();
+
+        // User one has received all the rewards and has no more previewed rewards.
+        assertEq(feesAndMevUserOne.balance, 2 ether);
+        assertEq(giantFeesAndMevPool.totalRewardsReceived(), 2 ether);
+        assertEq(
+            giantFeesAndMevPool.previewAccumulatedETH(
+                feesAndMevUserOne,
+                new address[](0),
+                new LPToken[][](0)),
+                0);
+
+        // Check the claimed[] value for user 1. It is correct.
+        assertEq(
+            giantFeesAndMevPool.claimed(feesAndMevUserOne, address(giantFeesAndMevPool.lpTokenETH())),
+            2 ether);
+
+        // User one transfers half their giant tokens to user 2.
+        vm.startPrank(feesAndMevUserOne);
+        giantFeesAndMevPool.lpTokenETH().transfer(feesAndMevUserTwo, 4 ether);
+        vm.stopPrank();
+
+        // After the tokens have been transferred to user 2, user 1's claimed[] remains
+        // unchanged - and is higher than the accumulated payout per share for user 1's
+        // current number of shares.
+        assertEq(
+            giantFeesAndMevPool.claimed(feesAndMevUserOne, address(giantFeesAndMevPool.lpTokenETH())),
+            2 ether);
+
+        // With this incorrect value of claimed[] causing a subtraction underflow, user one
+        // cannot preview accumulated eth or perform any action that attempts to claim their
+        // rewards such as transferring their tokens.
+        vm.startPrank(feesAndMevUserOne);
+        vm.expectRevert();
+        giantFeesAndMevPool.previewAccumulatedETH(
+            feesAndMevUserOne,
+            new address[](0),
+            new LPToken[][](0));
+
+        console.log("the revert expected now");
+        GiantLP token = giantFeesAndMevPool.lpTokenETH();
+        vm.expectRevert();
+        token.transfer(feesAndMevUserTwo, 1 ether);
+        vm.stopPrank();
+
+        // Add 1 eth rewards to manager2's staking funds vault.
+        vm.deal(address(manager2.stakingFundsVault()), 2 ether);
+
+        // User 2 claims rewards into the giant pool and obtains its 1/2 share.
+        vm.startPrank(feesAndMevUserTwo);
+        giantFeesAndMevPool.claimRewards(
+            feesAndMevUserTwo,
+            getAddressArrayFromValues(address(manager2.stakingFundsVault())),
+            blsPubKeyTwoInput);
+        vm.stopPrank();
+        assertEq(feesAndMevUserTwo.balance, 1 ether);
+
+        // At this point, user 1 ought to have accumulated 1 ether from the rewards,
+        // however accumulated eth is listed as 0.
+        // The reason is that when the giant pool tokens were transferred to
+        // user two, the claimed[] value for user one was left unchanged.
+        assertEq(
+            giantFeesAndMevPool.previewAccumulatedETH(
+                feesAndMevUserOne,
+                new address[](0),
+                new LPToken[][](0)),
+                0);
+
+        // The pool has received 4 eth rewards and paid out 3, but no users
+        // are listed as having accumulated the eth. It is orphaned.
+        assertEq(giantFeesAndMevPool.totalRewardsReceived(), 4 ether);
+        assertEq(giantFeesAndMevPool.totalClaimed(), 3 ether);
+
+        assertEq(
+            giantFeesAndMevPool.previewAccumulatedETH(
+                feesAndMevUserTwo,
+                new address[](0),
+                new LPToken[][](0)),
+                0);
+
+    }
+
 }
\ No newline at end of file

```
## Tools Used

## Recommended Mitigation Steps
Reduce claimed[] when necessary on the from side when GiantMevAndFeesPool tokens are transferred. Alternatively, claimed[] could be calculated on a per share basis rather than a total basis in order to simplify some of the adjustments that must be made in the code for claimed[].