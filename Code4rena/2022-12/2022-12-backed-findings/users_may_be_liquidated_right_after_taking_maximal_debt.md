## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- H-04

# [Users may be liquidated right after taking maximal debt](https://github.com/code-423n4/2022-12-backed-findings/issues/190) 

# Lines of code

https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L471
https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L317


# Vulnerability details

## Impact
Since there's no gap between the maximal LTV and the liquidation LTV, user positions may be liquidated as soon as maximal debt is taken, without leaving room for collateral and Papr token prices fluctuations. Users have no chance to add more collateral or reduce debt before being liquidated. This may eventually create more uncovered and bad debt for the protocol.
## Proof of Concept
The protocol allows users to take debt up to the maximal debt, including it ([PaprController.sol#L471](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L471)):
```solidity
if (newDebt > max) revert IPaprController.ExceedsMaxDebt(newDebt, max);
```

However, a positions becomes liquidable as soon as user's debt reaches user's maximal debt ([PaprController.sol#L317](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L317)):
```solidity
if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
    revert IPaprController.NotLiquidatable();
}
```

Moreover, the same maximal debt calculation is used during borrowing and liquidating, with the same maximal LTV ([PaprController.sol#L556-L559](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L556-L559)):
```solidity
function _maxDebt(uint256 totalCollateraValue, uint256 cachedTarget) internal view returns (uint256) {
    uint256 maxLoanUnderlying = totalCollateraValue * maxLTV;
    return maxLoanUnderlying / cachedTarget;
}
```

Even though different price kinds are used during borrowing and liquidations ([LOWER during borrowing](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L467), [TWAP during liquidations](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L316)), the price can in fact match ([ReservoirOracleUnderwriter.sol#L11](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/ReservoirOracleUnderwriter.sol#L11)):
```solidity
/// @dev LOWER is the minimum of SPOT and TWAP
```
Which means that the difference in prices doesn't always create a gap in maximal and liquidation LTVs.

The combination of these factors allows users to take maximal debts and be liquidated immediately, in the same block. Since liquidations are not beneficial for lending protocols, such heavy penalizing of users may harm the protocol and increase total uncovered debt, and potentially lead to a high bad debt.

```solidity
// test/paprController/IncreaseDebt.t.sol

event RemoveCollateral(address indexed account, ERC721 indexed collateralAddress, uint256 indexed tokenId);

function testIncreaseDebtAndBeLiquidated_AUDIT() public {
    vm.startPrank(borrower);
    nft.approve(address(controller), collateralId);
    IPaprController.Collateral[] memory c = new IPaprController.Collateral[](1);
    c[0] = collateral;
    controller.addCollateral(c);

    // Calculating the max debt for the borrower.
    uint256 maxDebt = controller.maxDebt(1 * oraclePrice);

    // Taking the maximal debt.
    vm.expectEmit(true, true, false, true);
    emit IncreaseDebt(borrower, collateral.addr, maxDebt);
    controller.increaseDebt(borrower, collateral.addr, maxDebt, oracleInfo);
    vm.stopPrank();

    // Making a TWAP price that's identical to the LOWER one.
    priceKind = ReservoirOracleUnderwriter.PriceKind.TWAP;
    ReservoirOracleUnderwriter.OracleInfo memory twapOracleInfo = _getOracleInfoForCollateral(nft, underlying);

    // The borrower is liquidated in the same block.
    vm.expectEmit(true, true, false, false);
    emit RemoveCollateral(borrower, collateral.addr, collateral.id);
    controller.startLiquidationAuction(borrower, collateral, twapOracleInfo);
}
```

## Tools Used
Manual review
## Recommended Mitigation Steps
Consider adding a liquidation LTV that's bigger than the maximal borrow LTV; positions can only be liquidated after reaching the liquidation LTV. This will create a room for price fluctuations and let users increase their collateral or decrease debt before being liquidating.
Alternatively, consider liquidating positions only after their debt has increased the maximal one:
```diff
--- a/src/PaprController.sol
+++ b/src/PaprController.sol
@@ -314,7 +314,7 @@ contract PaprController is

         uint256 oraclePrice =
             underwritePriceForCollateral(collateral.addr, ReservoirOracleUnderwriter.PriceKind.TWAP, oracleInfo);
-        if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
+        if (info.debt <= _maxDebt(oraclePrice * info.count, cachedTarget)) {
             revert IPaprController.NotLiquidatable();
         }

```