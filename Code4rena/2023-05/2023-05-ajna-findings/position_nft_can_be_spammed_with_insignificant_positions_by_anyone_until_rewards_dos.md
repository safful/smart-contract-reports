## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [Position NFT can be spammed with insignificant positions by anyone until rewards DoS](https://github.com/code-423n4/2023-05-ajna-findings/issues/488) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L170-L216
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L466-L485


# Vulnerability details

## Impact
The [PositionManager.memorializePositions(params_)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L170-L216) method can be called by **anyone** (per design, see 3rd party test cases) and allows **insignificantly** small (any value > 0) positions to be attached to **anyone** else's positions NFT, see PoC. As a result, the `positionIndexes[params_.tokenId]` storage array for an NFT with given token ID can be spammed with positions without the NFT owner's consent.  

Therefore, the [PositionManager.getPositionIndexesFiltered(tokenId_)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L466-L485) method might exceed the block gas limit when iterating the `positionIndexes[tokenId_]` storage array. However, the [RewardsManager.calculateRewards(...)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L325-L349) and [RewardsManager._calculateAndClaimRewards(...)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L384-L414) methods rely on the aforementioned method to succeed in order to calculate and pay rewards.

All in all, a griefer can spam anyone's position NFT with insignificant positions until the rewards mechanism fails for the NFT owner due to DoS (gas limit). Side note: A position NFT also cannot be burned as long as such insignificant positions are attached to it, see [PositionManager.burn(...)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L142-L154).


## Proof of Concept

The following *diff* is based on the existing test case `testMemorializePositions` in `PositionManager.t.sol` and demonstrates that insignificant positions can be attached by anyone.
```diff
diff --git a/ajna-core/tests/forge/unit/PositionManager.t.sol b/ajna-core/tests/forge/unit/PositionManager.t.sol
index bf3aa40..56c85d1 100644
--- a/ajna-core/tests/forge/unit/PositionManager.t.sol
+++ b/ajna-core/tests/forge/unit/PositionManager.t.sol
@@ -122,6 +122,7 @@ contract PositionManagerERC20PoolTest is PositionManagerERC20PoolHelperContract
      */
     function testMemorializePositions() external {
         address testAddress = makeAddr("testAddress");
+        address otherAddress = makeAddr("otherAddress");
         uint256 mintAmount  = 10000 * 1e18;

         _mintQuoteAndApproveManagerTokens(testAddress, mintAmount);
@@ -134,17 +135,17 @@ contract PositionManagerERC20PoolTest is PositionManagerERC20PoolHelperContract

         _addInitialLiquidity({
             from:   testAddress,
-            amount: 3_000 * 1e18,
+            amount: 1, //3_000 * 1e18,
             index:  indexes[0]
         });
         _addInitialLiquidity({
             from:   testAddress,
-            amount: 3_000 * 1e18,
+            amount: 1, //3_000 * 1e18,
             index:  indexes[1]
         });
         _addInitialLiquidity({
             from:   testAddress,
-            amount: 3_000 * 1e18,
+            amount: 1, // 3_000 * 1e18,
             index:  indexes[2]
         });

@@ -165,17 +166,20 @@ contract PositionManagerERC20PoolTest is PositionManagerERC20PoolHelperContract

         // allow position manager to take ownership of the position
         uint256[] memory amounts = new uint256[](3);
-        amounts[0] = 3_000 * 1e18;
-        amounts[1] = 3_000 * 1e18;
-        amounts[2] = 3_000 * 1e18;
+        amounts[0] = 1; //3_000 * 1e18;
+        amounts[1] = 1; //3_000 * 1e18;
+        amounts[2] = 1; //3_000 * 1e18;
         _pool.increaseLPAllowance(address(_positionManager), indexes, amounts);

         // memorialize quote tokens into minted NFT
+        changePrank(otherAddress); // switch other address (not owner of NFT)
         vm.expectEmit(true, true, true, true);
-        emit TransferLP(testAddress, address(_positionManager), indexes, 9_000 * 1e18);
+        emit TransferLP(testAddress, address(_positionManager), indexes, 3 /*9_000 * 1e18*/);
         vm.expectEmit(true, true, true, true);
         emit MemorializePosition(testAddress, tokenId, indexes);
-        _positionManager.memorializePositions(memorializeParams);
+        _positionManager.memorializePositions(memorializeParams);  // switch back to test address (owner of NFT)
+        changePrank(testAddress);
+

         // check memorialization success
         uint256 positionAtPriceOneLP = _positionManager.getLP(tokenId, indexes[0]);

```

## Tools Used
VS Code, Foundry

## Recommended Mitigation Steps
Requiring that The [PositionManager.memorializePositions(params_)](https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L170-L216) can only be called by the NFT owner or anyone who has approval would help but break the 3rd party test cases.  
Alternatively, one could enforce a minimum position value to make this griefing attack extremely unattractive.



## Assessed type

DoS