## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- H-03

# [Improper validations in Clearinghouse. possible to lock collateral NFT in contract.](https://github.com/code-423n4/2023-01-astaria-findings/issues/521) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L114-L178


# Vulnerability details

When a borrower does not return the borrowed funds on time, a liquidator can trigger a liquidation.
In that case the collateral NFT will be listed in an Seaport dutch auction.
The auction requests settlementToken and a fake ClearingHouse NFT.
When a buyer bids enough of the settlementToken, openSea auction will accept the offer, transfer the NFT from ClearingHouse to bidder, move settlementToken from bidder to ClearingHouse, and 'transfers' the fake clearinghouse NFT to clearinghouse. This call to [ClearingHouse.safeTransferFrom](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L169-L178) triggers the further processing of the liquidation. It will pay the debt with the funds received from Seaport, and delete data from LienToken and CollateralToken for this collateral NFT. 

The problem is that the `ClearingHouse.safeTransferFrom` can be called by anyone and assumes valid call parameters. One of the parameters `identifier` is used to pass the paymentToken address. This can easily be modified to let the contract accept any ERC20 token as `paymentToken` to payoff the debt.
This allows a malicious actor to lock a user's collateral NFT and cancel the auction. 
This could be misused to completely block any liquidatons.

The steps to reproduce: 

    1. Normal flow: borrow funds via requestLienPosition
    2. borrowed funds are not paid back before stack.point.end 
    3. liquidator calls AstariaRouter.liquidate(...)

    At this time a Seaport auction is initiated and CollateralToken state for this collateralID is updated to be in auction. 
    All fine so far.

    4. An evil actor can now call ClearingHouse.safeTransferFrom with dummy data and a dummy ERC20  token address as paymentToken

After this call, the Collateral NFT will still be in de ClearingHouse contract, but references to the NFT are cleaned up from both CollateralToken and LienToken.
This results in the NFT being locked in the contract without any way to get it out.

## Technical details: 
When a liquidation is started, [_generateValidOrderParameters](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L425) is called to generate the Seaport order params. It sets [settlementToken as the identifierOrCriteria](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L456).

ClearingHouse assumes that `safeTransferFrom` will only be called by Seaport after a succesful auction, and assumes the identifier is the `settlementToken` value that was set for the order.
The _execute function is called, which [converts the identifier parameter to a paymentToken address](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L123) and checks if the received amount of paymentToken is >= the expected auction currentOfferPrice it accepts the call and moves the token balance to the correct addresses.
It then calls [LienToken.payDebtViaClearingHouse](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L497-L510), passing the fake `paymentToken` as a parameter.
Lientoken contract also does not verify if the token is ineed the correct settlementToken and pays off the debt with the fake token balance.
It then deletes the [collateralStateHash](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L509) for the collateralId, removing the stack state.

After that,  `CollateralToken.settleAuction` is called, which [burns the token](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L538) for collateralId and [deletes idToUnderlying](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L537) and [collateralIdToAuction](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L544) for this collateralId.

We now have a state where the collateral NFT is in the ClearingHouse contract, but all actions are made impossible, because the state in the token contracts are removed.
`CollateralToken.ownerOf(collateralID)` reverts because the entry in `s.idToUnderlying` was removed.
This causes `releaseToAddress()` and `flashAction()` to fail.
`liquidatorNFTClaim` fails because `s.collateralIdToAuction` was cleared.
Trying to create a new Lien alos fails as that calls `CollateralToken.ownerOf(collateralID)`.


## Proof of concept.

To test the scenario, I have modified the testLiquidationNftTransfer test in AstariaTest.t.sol

```diff
diff --git a/src/test/AstariaTest.t.sol b/src/test/AstariaTest.t.sol
index c7ce162..bfaeca6 100644
--- a/src/test/AstariaTest.t.sol
+++ b/src/test/AstariaTest.t.sol
@@ -18,6 +18,7 @@ import "forge-std/Test.sol";
 import {Authority} from "solmate/auth/Auth.sol";
 import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
 import {MockERC721} from "solmate/test/utils/mocks/MockERC721.sol";
+import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";
 import {
   MultiRolesAuthority
 } from "solmate/auth/authorities/MultiRolesAuthority.sol";
@@ -1030,8 +1031,19 @@ contract AstariaTest is TestHelpers {
       uint8(0)
     );
     vm.stopPrank();
-    uint256 bid = 100 ether;
-    _bid(Bidder(bidder, bidderPK), listedOrder, bid);
+
+    uint256 collateralId = tokenContract.computeId(tokenId);
+    ClearingHouse CH = COLLATERAL_TOKEN.getClearingHouse(collateralId);
+
+    // create a worthless token and send it to the clearinghouse
+    MockERC20 worthlessToken = new MockERC20("TestToken","TST",18);
+    worthlessToken.mint(address(CH),550 ether);
+
+    // call safeTransferFrom on clearinghouse with the worthless token as paymentToken
+    // thiss will trigger the cleaning up after succesful auction
+    uint256 tokenAsInt = uint256(uint160(address(worthlessToken)));
+    bytes memory emptyBytes;
+    CH.safeTransferFrom(address(0),address(bidder),tokenAsInt,0,emptyBytes);

     // assert the bidder received the NFT
     assertEq(nft.ownerOf(tokenId), bidder, "Bidder did not receive NFT");
```

After this, the [assert the bidder received the NFT](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/test/AstariaTest.t.sol#L1037) test will fail, as the NFT is not moved.
But the state of the CollateralToken and LienToken contracts is updated. 

## Tools Used

Manual audit, forge

## Recommended Mitigation Steps
Minimal fix would be to check either check if the supplied paymentToken matches the expected paymentToken, or ignore the parameter alltogether and use the paymentToken from the contract. Other option is to restrict calls to the function to whitelisted addresses (OpenSea controler and conduit ).

In the current setup, there is no easy way for the ClearingHouse to access information about the settlementToken.
It could be added to the AuctionData struct:

```diff
diff --git a/src/interfaces/ILienToken.sol b/src/interfaces/ILienToken.sol
index 964caa2..06433c0 100644
--- a/src/interfaces/ILienToken.sol
+++ b/src/interfaces/ILienToken.sol
@@ -238,6 +238,7 @@ interface ILienToken is IERC721 {
     uint48 startTime;
     uint48 endTime;
     address liquidator;
+    address settlementToken;
     AuctionStack[] stack;
   }
```

and then addi it in LienToken
```diff
diff --git a/src/LienToken.sol b/src/LienToken.sol
index 631ac02..372e197 100644
--- a/src/LienToken.sol
+++ b/src/LienToken.sol
@@ -340,6 +340,8 @@ contract LienToken is ERC721, ILienToken, AuthInitializable {
       .liquidationInitialAsk
       .safeCastTo88();
     auctionData.endAmount = uint88(1000 wei);
+    auctionData.settlementToken = stack[0].lien.token; 
+
     s.COLLATERAL_TOKEN.getClearingHouse(collateralId).setAuctionData(
       auctionData
     );
```

In the ClearingHouse it can be used directly, ignoring the supplied parameter: 
```diff
diff --git a/src/ClearingHouse.sol b/src/ClearingHouse.sol
index 5c2a400..d305ff5 100644
--- a/src/ClearingHouse.sol
+++ b/src/ClearingHouse.sol
@@ -120,7 +120,7 @@ contract ClearingHouse is AmountDeriver, Clone, IERC1155, IERC721Receiver {
     IAstariaRouter ASTARIA_ROUTER = IAstariaRouter(_getArgAddress(0)); // get the router from the immutable arg

     ClearingHouseStorage storage s = _getStorage();
-    address paymentToken = bytes32(encodedMetaData).fromLast20Bytes();
+    address paymentToken = s.auctionStack.settlementToken;

     uint256 currentOfferPrice = _locateCurrentAmount({
       startAmount: s.auctionStack.startAmount,
```
or used to check the supplied paramameter
```diff
diff --git a/src/ClearingHouse.sol b/src/ClearingHouse.sol
index 5c2a400..5a79184 100644
--- a/src/ClearingHouse.sol
+++ b/src/ClearingHouse.sol
@@ -121,6 +121,7 @@ contract ClearingHouse is AmountDeriver, Clone, IERC1155, IERC721Receiver {

     ClearingHouseStorage storage s = _getStorage();
     address paymentToken = bytes32(encodedMetaData).fromLast20Bytes();
+    require(paymentToken == s.auctionStack.settlementToken);

     uint256 currentOfferPrice = _locateCurrentAmount({
       startAmount: s.auctionStack.startAmount,
```

With this step added, it would still be possible to lock the NFT in the contract, but this time that will only succeed when the requested auction amount is paid. So in that case it would be more logical to simply bid on OpenSea and also get the NFT instead of paying the tokens just to lock it.
