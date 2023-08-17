## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-06

# [`PaprController` pays swap fee in `buyAndReduceDebt`, not user](https://github.com/code-423n4/2022-12-backed-findings/issues/196) 

# Lines of code

https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L226


# Vulnerability details

## Impact
Since `PaprController` is not designed to hold any underlying tokens, calling `buyAndReduceDebt` with a swap fee set will result in a revert. The function can also be used to transfer out any underlying tokens sent to the contract mistakenly.
## Proof of Concept
`PaprController` implements the `buyAndReduceDebt` function, which allows users to buy Papr tokens for underlying tokens and burn them to reduce their debt ([PaprController.sol#L208](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L208)). Optionally, the function allows the caller to specify a swap fee: a fee that's collected from the caller. However, in reality, the fee is collected from `PaprController` itself: `transfer` instead of `transferFrom` is called on the underlying token ([PaprController.sol#L225-L227](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/src/PaprController.sol#L225-L227)):
```solidity
if (hasFee) {
    underlying.transfer(params.swapFeeTo, amountIn * params.swapFeeBips / BIPS_ONE);
}
```

This scenario is covered by the `testBuyAndReduceDebtReducesDebt` test ([BuyAndReduceDebt.t.sol#L12](https://github.com/with-backed/papr/blob/9528f2711ff0c1522076b9f93fba13f88d5bd5e6/test/paprController/BuyAndReduceDebt.t.sol#L12)), however the fee is not actually set in the test:
```solidity
// Fee is initialized but not set.
uint256 fee;
underlying.approve(address(controller), underlyingOut + underlyingOut * fee / 1e4);
swapParams = IPaprController.SwapParams({
    amount: underlyingOut,
    minOut: 1,
    sqrtPriceLimitX96: _maxSqrtPriceLimit({sellingPAPR: false}),
    swapFeeTo: address(5),
    swapFeeBips: fee
});
```

If fee is set in the test, the test wil revert with an "Arithmetic over/underflow" error:
```diff
--- a/test/paprController/BuyAndReduceDebt.t.sol
+++ b/test/paprController/BuyAndReduceDebt.t.sol
@@ -26,7 +26,7 @@ contract BuyAndReduceDebt is BasePaprControllerTest {
         IPaprController.VaultInfo memory vaultInfo = controller.vaultInfo(borrower, collateral.addr);
         assertEq(vaultInfo.debt, debt);
         assertEq(underlyingOut, underlying.balanceOf(borrower));
-        uint256 fee;
+        uint256 fee = 1e3;
         underlying.approve(address(controller), underlyingOut + underlyingOut * fee / 1e4);
         swapParams = IPaprController.SwapParams({
             amount: underlyingOut,
```

## Tools Used
Manual review
## Recommended Mitigation Steps
Consider this change:
```diff
--- a/src/PaprController.sol
+++ b/src/PaprController.sol
@@ -223,7 +223,7 @@ contract PaprController is
         );

         if (hasFee) {
-            underlying.transfer(params.swapFeeTo, amountIn * params.swapFeeBips / BIPS_ONE);
+            underlying.safeTransferFrom(msg.sender, params.swapFeeTo, amountIn * params.swapFeeBips / BIPS_ONE);
         }

         _reduceDebt({account: account, asset: collateralAsset, burnFrom: msg.sender, amount: amountOut});
```