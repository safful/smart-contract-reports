## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-25

# [`UlyssesToken` asset ID accounting error](https://github.com/code-423n4/2023-05-maia-findings/issues/275) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L72


# Vulnerability details

## Impact
Asset IDs in the `UlyssesToken` contract are **1-based**, see [L49 in UlyssesToken.addAsset(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L49) and [L55 in ERC4626MultiToken.constructor(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/ERC4626MultiToken.sol#L55) of the parent contract.  
However, when removing an asset from the `UlyssesToken` contract, the last added asset gets the **0-based** ID of the removed asset, see [L72 in UlyssesToken.removeAsset(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L72).  

This leads to the following consequences:
1. Duplicate IDs when removing an asset.  
Example: We have assets with IDs `1,2,3,4`. Next, asset with ID=2 is removed. Now we have assets with IDs `1,1,3` because the last asset with ID=4 gets ID=2-1=1.
2. Last asset cannot be removed after removing first asset.  
Example: Once the first asset with ID=1 is removed, the last asset gets ID=0 instead of ID=1. When trying to remove the last asset [L62 in UlyssesToken.removeAsset(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L62) will **revert** due to underflow.
3. Last asset can be added a second time after removing first asset.  
Example: Once the first asset with ID=1 is removed, the last asset gets ID=0 instead of ID=1. When trying to add the last asset again [L45 in UlyssesToken.addAsset(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L45) will **not revert** since ID=0 indicates that the asset wasn't added yet. Therefore, the underlying vault can contain the same token twice with **different** weights.

In conclusion, the asset accounting of the `UlyssesToken` contract is broken after removing an asset (especially the first one). This was also highlighted as a special area of concern in the audit details: `ulysses AMM and token accounting`

## Proof of Concept

The above issues are demostrated by the new test cases `test_UlyssesTokenAddAssetTwice` and `test_UlyssesTokenRemoveAssetFail`, just apply the *diff* below and run the tests with `forge test --match-test test_UlyssesToken`.
```diff
diff --git a/test/ulysses-amm/UlyssesTokenTest.t.sol b/test/ulysses-amm/UlyssesTokenTest.t.sol
index bdb4a7d..dcf6d45 100644
--- a/test/ulysses-amm/UlyssesTokenTest.t.sol
+++ b/test/ulysses-amm/UlyssesTokenTest.t.sol
@@ -3,6 +3,7 @@ pragma solidity >=0.8.0 <0.9.0;

 import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";
 import {UlyssesToken} from "@ulysses-amm/UlyssesToken.sol";
+import {IUlyssesToken} from "@ulysses-amm/interfaces/IUlyssesToken.sol";

 import {UlyssesTokenHandler} from "@test/test-utils/invariant/handlers/UlyssesTokenHandler.t.sol";

@@ -29,4 +30,28 @@ contract InvariantUlyssesToken is UlyssesTokenHandler {
         _vaultMayBeEmpty = true;
         _unlimitedAmount = false;
     }
+
+    function test_UlyssesTokenRemoveAssetFail() public  {
+        UlyssesToken token = UlyssesToken(_vault_);
+
+        // remove first asset with ID=1
+        token.removeAsset(_underlyings_[0]);
+        // due to accounting error, last asset now has ID=0 instead of ID=1
+
+        // remove last asset --> underflow error due to ID=0
+        token.removeAsset(_underlyings_[NUM_ASSETS - 1]);
+    }
+
+    function test_UlyssesTokenAddAssetTwice() public  {
+        UlyssesToken token = UlyssesToken(_vault_);
+
+        // remove first asset with ID=1
+        token.removeAsset(_underlyings_[0]);
+        // due to accounting error, last asset now has ID=0 instead of ID=1
+
+        // add last asset again --> doesn't revert since it "officially" doesn't exist due to ID=1
+        vm.expectRevert(IUlyssesToken.AssetAlreadyAdded.selector);
+        token.addAsset(_underlyings_[NUM_ASSETS - 1], 1);
+    }
+
 }

```

We can see that adding the last asset again does **not revert** but trying to remove it still **fails**:
```
Encountered 2 failing tests in test/ulysses-amm/UlyssesTokenTest.t.sol:InvariantUlyssesToken
[FAIL. Reason: Call did not revert as expected] test_UlyssesTokenAddAssetTwice() (gas: 169088)
[FAIL. Reason: Arithmetic over/underflow] test_UlyssesTokenRemoveAssetFail() (gas: 137184)
```

## Tools Used
VS Code, Foundry, MS Excel

## Recommended Mitigation Steps

Fortunately, all of the above issues can be easily fixed by using an **1-based** asset ID in [L72 of UlyssesToken.removeAsset(...)](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesToken.sol#L72):
```diff
diff --git a/src/ulysses-amm/UlyssesToken.sol b/src/ulysses-amm/UlyssesToken.sol
index 552a125..0937e9f 100644
--- a/src/ulysses-amm/UlyssesToken.sol
+++ b/src/ulysses-amm/UlyssesToken.sol
@@ -69,7 +69,7 @@ contract UlyssesToken is ERC4626MultiToken, Ownable, IUlyssesToken {

         address lastAsset = assets[newAssetsLength];

-        assetId[lastAsset] = assetIndex;
+        assetId[lastAsset] = assetIndex + 1;
         assets[assetIndex] = lastAsset;
         weights[assetIndex] = weights[newAssetsLength];

```

After applying the recommended fix, both new test cases pass:
```
[PASS] test_UlyssesTokenAddAssetTwice() (gas: 122911)
[PASS] test_UlyssesTokenRemoveAssetFail() (gas: 134916)
```



## Assessed type

Under/Overflow