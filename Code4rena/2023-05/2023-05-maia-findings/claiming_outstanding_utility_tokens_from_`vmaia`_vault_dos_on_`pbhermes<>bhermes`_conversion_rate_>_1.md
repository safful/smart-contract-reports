## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-23

# [Claiming outstanding utility tokens from `vMaia` vault DoS on `pbHermes<>bHermes` conversion rate > 1](https://github.com/code-423n4/2023-05-maia-findings/issues/470) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L215-L229
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/vMaia.sol#L66-L88


# Vulnerability details

## Impact
Once a user deposits `Maia` ERC-20 tokens into the `vMaia` ERC-4626 vault, he is eligible to claim 3 kinds of utility tokens, bHermes Weight & Governance and Maia Governance (pbHermes, partner governance), via the [ERC4626PartnerManager.claimOutstanding()](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L89-L97) method (`ERC4626PartnerManager` is base of `vMaia` contract).  
The conversion rate between the utility tokens and [vMaia tokens minted on deposit](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L235-L246) can be increased (and **only** increased) by the contract owner via the [ERC4626PartnerManager.increaseConversionRate(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L215-L229) method.  

However, the [checkWeight, checkGovernance & checkPartnerGovernance](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/vMaia.sol#L66-L88) modifiers in the `vMaia` contract do not account for this conversion rate and therefore implicity only allow a conversion rate of 1.  

As a consequence, as soon as the conversion rate is increased to > 1, a call to [ERC4626PartnerManager.claimOutstanding()](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L89-L97) will inevitably revert due to subsequent calls to the above modifiers. Since the conversion rate can [only be increased](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L217) and the `vMaia` vault contract is not upgradeable, the `claimOutstanding()` method is subject to **permanent DoS**.  
Of course, the user can still claim a reduced amount of utility tokens (according to a conversion rate of 1) via the [PartnerUtilityManager.claimMultipleAmounts(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/PartnerUtilityManager.sol#L115-L124) method (`PartnerUtilityManager` is base of `ERC4626PartnerManager` contract), but this still implies a loss of assets for the user since not all utility tokens he is eligible for can be claimed. Furthermore, this workaround doesn't help when the user is a contract which implemented a call to the `claimOutstanding()` method.

## Proof of Concept

The following PoC demonstrates the above DoS when trying to claim the utility tokens with increased conversion rate, just apply the *diff* below and run the test cases with `forge test -vv --match-test testDepositMaia`:
```diff
diff --git a/test/maia/vMaiaTest.t.sol b/test/maia/vMaiaTest.t.sol
index 6efabc5..499abb6 100644
--- a/test/maia/vMaiaTest.t.sol
+++ b/test/maia/vMaiaTest.t.sol
@@ -7,9 +7,11 @@ import {SafeTransferLib} from "solady/utils/SafeTransferLib.sol";

 import {vMaia, PartnerManagerFactory, ERC20} from "@maia/vMaia.sol";
 import {IBaseVault} from "@maia/interfaces/IBaseVault.sol";
+import {IERC4626PartnerManager} from "@maia/interfaces/IERC4626PartnerManager.sol";
 import {MockVault} from "./mock/MockVault.t.sol";

 import {bHermes} from "@hermes/bHermes.sol";
+import {IUtilityManager} from "@hermes/interfaces/IUtilityManager.sol";

 import {DateTimeLib} from "solady/utils/DateTimeLib.sol";

@@ -47,7 +49,7 @@ contract vMaiaTest is DSTestPlus {
             "vMAIA",
             address(bhermes),
             address(vault),
-            address(0)
+            address(this) // set owner to allow call to 'increaseConversionRate'
         );
     }

@@ -86,6 +88,33 @@ contract vMaiaTest is DSTestPlus {
         assertEq(vmaia.balanceOf(address(this)), amount);
     }

+    function testDepositMaiaClaimDoS() public {
+        testDepositMaia();
+
+        // increase 'pbHermes<>bHermes' conversion rate
+        vmaia.increaseConversionRate(bHermesRate * 2);
+
+        // claim utility tokens DoS
+        hevm.expectRevert(IUtilityManager.InsufficientShares.selector);
+        vmaia.claimOutstanding();
+
+        // cannot undo conversion rate -> claimOutstanding() method is broken forever
+        hevm.expectRevert(IERC4626PartnerManager.InvalidRate.selector);
+        vmaia.increaseConversionRate(bHermesRate);
+    }
+
+    function testDepositMaiaClaimSuccess() public {
+        testDepositMaia();
+
+        vmaia.claimOutstanding();
+
+        // got utility tokens as expected
+        assertGt(vmaia.bHermesToken().gaugeWeight().balanceOf(address(this)), 0);
+        assertGt(vmaia.bHermesToken().governance().balanceOf(address(this)), 0);
+        assertGt(vmaia.partnerGovernance().balanceOf(address(this)), 0);
+    }
+
+
     function testDepositMaiaAmountFail() public {
         assertEq(vmaia.bHermesRate(), bHermesRate);

```

## Tools Used
VS Code, Foundry

## Recommended Mitigation Steps

Simply remove the incorrect [checkWeight, checkGovernance & checkPartnerGovernance](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/vMaia.sol#L66-L88) modifiers from the `vMaia` contract, since the [correct modifiers](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L293-L323), which account for the conversion rate, are already implemented in the `ERC4626PartnerManager` contract.

```diff
diff --git a/src/maia/vMaia.sol b/src/maia/vMaia.sol
index 3aa70cf..5ee6f66 100644
--- a/src/maia/vMaia.sol
+++ b/src/maia/vMaia.sol
@@ -59,34 +59,6 @@ contract vMaia is ERC4626PartnerManager {
         currentMonth = DateTimeLib.getMonth(block.timestamp);
     }

-    /*///////////////////////////////////////////////////////////////
-                            MODIFIERS
-    //////////////////////////////////////////////////////////////*/
-
-    /// @dev Checks available weight allows for the call.
-    modifier checkWeight(uint256 amount) virtual override {
-        if (balanceOf[msg.sender] < amount + userClaimedWeight[msg.sender]) {
-            revert InsufficientShares();
-        }
-        _;
-    }
-
-    /// @dev Checks available governance allows for the call.
-    modifier checkGovernance(uint256 amount) virtual override {
-        if (balanceOf[msg.sender] < amount + userClaimedGovernance[msg.sender]) {
-            revert InsufficientShares();
-        }
-        _;
-    }
-
-    /// @dev Checks available partner governance allows for the call.
-    modifier checkPartnerGovernance(uint256 amount) virtual override {
-        if (balanceOf[msg.sender] < amount + userClaimedPartnerGovernance[msg.sender]) {
-            revert InsufficientShares();
-        }
-        _;
-    }
-
     /// @dev Boost can't be claimed; does not fail. It is all used by the partner vault.
     function claimBoost(uint256 amount) public override {}

```



## Assessed type

Invalid Validation