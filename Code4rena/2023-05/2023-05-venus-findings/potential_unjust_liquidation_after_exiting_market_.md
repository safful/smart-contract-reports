## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [Potential Unjust Liquidation After Exiting Market ](https://github.com/code-423n4/2023-05-venus-findings/issues/309) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L424-L526


# Vulnerability details

## Impact

Users might face unjust liquidation of their assets even after exiting a particular market. This could lead to potential financial losses for users, and it might undermine the trust and reputation of the platform. 

## Proof of Concept

Consider a user with the following financial status:

- Collateral: 1 Bitcoin (BTC), worth `$30,000`, and 10,000 USDT, worth `$10,000`.
- Outstanding loan: 1 Ethereum (ETH), worth $3,000.

1. The user decides to remove their risk from BTC volatility and exits the BTC market. As per the protocol's rules, exiting the market should remove BTC from their collateral.

2. Following the user's exit from the BTC market, a sharp rise in the ETH price occurs, and it surpasses $10,000.

3. Due to the increase in ETH price, the system identifies that the user's collateral (now only 10,000 USDT) is insufficient to cover their loan, leading to an insufficient collateralization rate.

4. Despite the user's exit from the BTC market, the system still triggers a liquidation process to liquidating BTC as collateral. 

In reality, if the BTC was still part of the user's collateral, the total collateral value would have been $40,000 ($30,000 from BTC and $10,000 from USDT). This total value would be sufficient to cover the ETH loan even with the price surge of ETH. Therefore, the user should not have faced liquidation.

This can be traced back to the missing membership check in `preLiquidateHook` function which does not consider if the user has exited the market or not.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L424-L526

```solidity=424
function preLiquidateHook(
        address vTokenBorrowed,
        address vTokenCollateral,
        address borrower,
        uint256 repayAmount,
        bool skipLiquidityCheck
    ) external override {
        // Pause Action.LIQUIDATE on BORROWED TOKEN to prevent liquidating it.
        // If we want to pause liquidating to vTokenCollateral, we should pause
        // Action.SEIZE on it
        _checkActionPauseState(vTokenBorrowed, Action.LIQUIDATE);

        oracle.updatePrice(vTokenBorrowed);
        oracle.updatePrice(vTokenCollateral);

        if (!markets[vTokenBorrowed].isListed) {
            revert MarketNotListed(address(vTokenBorrowed));
        }
        if (!markets[vTokenCollateral].isListed) {
            revert MarketNotListed(address(vTokenCollateral));
        }

        uint256 borrowBalance = VToken(vTokenBorrowed).borrowBalanceStored(borrower);

        /* Allow accounts to be liquidated if the market is deprecated or it is a forced liquidation */
        if (skipLiquidityCheck || isDeprecated(VToken(vTokenBorrowed))) {
            if (repayAmount > borrowBalance) {
                revert TooMuchRepay();
            }
            return;
        }

        /* The borrower must have shortfall and collateral > threshold in order to be liquidatable */
        AccountLiquiditySnapshot memory snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);

        if (snapshot.totalCollateral <= minLiquidatableCollateral) {
            /* The liquidator should use either liquidateAccount or healAccount */
            revert MinimalCollateralViolated(minLiquidatableCollateral, snapshot.totalCollateral);
        }

        if (snapshot.shortfall == 0) {
            revert InsufficientShortfall();
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint256 maxClose = mul_ScalarTruncate(Exp({ mantissa: closeFactorMantissa }), borrowBalance);
        if (repayAmount > maxClose) {
            revert TooMuchRepay();
        }
    }

    /**
     * @notice Checks if the seizing of assets should be allowed to occur
     * @param vTokenCollateral Asset which was used as collateral and will be seized
     * @param seizerContract Contract that tries to seize the asset (either borrowed vToken or Comptroller)
     * @param liquidator The address repaying the borrow and seizing the collateral
     * @param borrower The address of the borrower
     * @custom:error ActionPaused error is thrown if seizing this type of collateral is paused
     * @custom:error MarketNotListed error is thrown if either collateral or borrowed token is not listed
     * @custom:error ComptrollerMismatch error is when seizer contract or seized asset belong to different pools
     * @custom:access Not restricted
     */
    function preSeizeHook(
        address vTokenCollateral,
        address seizerContract,
        address liquidator,
        address borrower
    ) external override {
        // Pause Action.SEIZE on COLLATERAL to prevent seizing it.
        // If we want to pause liquidating vTokenBorrowed, we should pause
        // Action.LIQUIDATE on it
        _checkActionPauseState(vTokenCollateral, Action.SEIZE);

        if (!markets[vTokenCollateral].isListed) {
            revert MarketNotListed(vTokenCollateral);
        }

        if (seizerContract == address(this)) {
            // If Comptroller is the seizer, just check if collateral's comptroller
            // is equal to the current address
            if (address(VToken(vTokenCollateral).comptroller()) != address(this)) {
                revert ComptrollerMismatch();
            }
        } else {
            // If the seizer is not the Comptroller, check that the seizer is a
            // listed market, and that the markets' comptrollers match
            if (!markets[seizerContract].isListed) {
                revert MarketNotListed(seizerContract);
            }
            if (VToken(vTokenCollateral).comptroller() != VToken(seizerContract).comptroller()) {
                revert ComptrollerMismatch();
            }
        }

        // Keep the flywheel moving
        uint256 rewardDistributorsCount = rewardsDistributors.length;

        for (uint256 i; i < rewardDistributorsCount; ++i) {
            rewardsDistributors[i].updateRewardTokenSupplyIndex(vTokenCollateral);
            rewardsDistributors[i].distributeSupplierRewardToken(vTokenCollateral, borrower);
            rewardsDistributors[i].distributeSupplierRewardToken(vTokenCollateral, liquidator);
        }
    }
```


In essence, the user is punished for market volatility even after they have taken steps to protect themselves (by exiting the BTC market).

## Tools Used

ChatGPT 4

## Recommended Mitigation Steps

Update the `preLiquidateHook` function to check if a user has exited a market before proceeding with liquidation. 



## Assessed type

Invalid Validation