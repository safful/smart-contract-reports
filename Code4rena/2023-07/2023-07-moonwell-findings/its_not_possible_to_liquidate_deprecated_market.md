## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-14

# [Its not possible to liquidate deprecated market](https://github.com/code-423n4/2023-07-moonwell-findings/issues/67) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Comptroller.sol#L394


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
Its not possible to liquidate deprecated market
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Currently in the code that is a function `_setBorrowPaused` that paused pause borrowing. In [origin compound code](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/Comptroller.sol#L1452) `borrowGuardianPaused` is used to do liquidate markets that are bad. So now there is not way to get rid of bad markets.
```solidity
    function liquidateBorrowAllowed(
        address mTokenBorrowed,
        address mTokenCollateral,
        address liquidator,
        address borrower,
        uint repayAmount) override external view returns (uint) {
        // Shh - currently unused
        liquidator;

        if (!markets[mTokenBorrowed].isListed || !markets[mTokenCollateral].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }

        /* The borrower must have shortfall in order to be liquidatable */
        (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall == 0) {
            return uint(Error.INSUFFICIENT_SHORTFALL);
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint borrowBalance = MToken(mTokenBorrowed).borrowBalanceStored(borrower);
        uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
        if (repayAmount > maxClose) {
            return uint(Error.TOO_MUCH_REPAY);
        }

        return uint(Error.NO_ERROR);
    }

```
[src/core/Comptroller.sol#L394](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Comptroller.sol#L394)
Here is a list why liquidating deprecated markets can be necessary:
1) Deprecated markets may pose security risks, especially if they are no longer actively monitored or maintained. By liquidating these markets, the platform reduces the potential for vulnerabilities and ensures that user funds are not exposed to unnecessary risks.

2) Deprecated markets might require ongoing resources to maintain and support, including development efforts and infrastructure costs. By liquidating these markets, the platform can optimize resources and focus on more actively used markets and features.
3) Older markets might be based on legacy smart contracts that lack the latest security enhancements and improvements. Liquidating these markets allows the platform to migrate users to more secure and up-to-date contracts.

4) Deprecated markets may result in a fragmented user base, leading to reduced liquidity and trading activity. By consolidating users onto actively used markets, the platform can foster higher liquidity and better user engagement.

## Tools Used

## Recommended Mitigation Steps
I think compound have a way to liquidate deprecated markets for a safety reason, so it needs to be restored
```diff
    function liquidateBorrowAllowed(
        address mTokenBorrowed,
        address mTokenCollateral,
        address liquidator,
        address borrower,
        uint repayAmount) override external view returns (uint) {
        // Shh - currently unused
        liquidator;
        /* allow accounts to be liquidated if the market is deprecated */
+        if (isDeprecated(CToken(cTokenBorrowed))) {
+            require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
+            return uint(Error.NO_ERROR);
+        }
        if (!markets[mTokenBorrowed].isListed || !markets[mTokenCollateral].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }

        /* The borrower must have shortfall in order to be liquidatable */
        (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall == 0) {
            return uint(Error.INSUFFICIENT_SHORTFALL);
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint borrowBalance = MToken(mTokenBorrowed).borrowBalanceStored(borrower);
        uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
        if (repayAmount > maxClose) {
            return uint(Error.TOO_MUCH_REPAY);
        }

        return uint(Error.NO_ERROR);
    }
+    function isDeprecated(CToken cToken) public view returns (bool) {
+        return
+        markets[address(cToken)].collateralFactorMantissa == 0 &&
+        borrowGuardianPaused[address(cToken)] == true &&
+        cToken.reserveFactorMantissa() == 1e18
+        ;
+    }
    
```


## Assessed type

Error