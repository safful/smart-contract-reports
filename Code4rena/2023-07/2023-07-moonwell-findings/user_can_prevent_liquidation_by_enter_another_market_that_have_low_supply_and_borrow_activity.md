## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-10

# [User can prevent liquidation by enter another market that have low supply and borrow activity](https://github.com/code-423n4/2023-07-moonwell-findings/issues/239) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L1263-L1273
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L358
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L402


# Vulnerability details

## Impact
Users can prevent themselves from being liquidated by entering another market that has low supply/borrow activity or have low/volatile value compared to other markets, the detailed scenario will be explained in PoC.

## Proof of Concept

The scenario :
- Alice enter market A, supply and borrow in that market.
- Alice enter market B that have low supply and borrow or low value compared to market A.
- Alice call `_addReserves` in market B so that `totalReserves` is bigger than `totalCash` + `totalBorrows`.
- After some time Alice has shortfall (his borrow value bigger than supply value).
- Another user try to liquidate Alice and seize her market A collateral by calling `liquidateBorrow` :

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MErc20.sol#L139-L142

```solidity
    function liquidateBorrow(address borrower, uint repayAmount, MTokenInterface mTokenCollateral) override external returns (uint) {
        (uint err,) = liquidateBorrowInternal(borrower, repayAmount, mTokenCollateral);
        return err;
    }
```

It will eventually trigger comptroller's `liquidateBorrowAllowed` hook :

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L970

```solidity
    function liquidateBorrowFresh(address liquidator, address borrower, uint repayAmount, MTokenInterface mTokenCollateral) internal returns (uint, uint) {
        /* Fail if liquidate not allowed */
        uint allowed = comptroller.liquidateBorrowAllowed(address(this), address(mTokenCollateral), liquidator, borrower, repayAmount);
        if (allowed != 0) {
            return (failOpaque(Error.COMPTROLLER_REJECTION, FailureInfo.LIQUIDATE_COMPTROLLER_REJECTION, allowed), 0);
        }
...
}
```

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Comptroller.sol#L394-L424

Inside `liquidateBorrowAllowed` hook, it will eventually call `getHypotheticalAccountLiquidityInternal` that will loop trough Alice entered markets/assets and call `getAccountSnapshot` to get token balance, borrow, and exchangeRate : 

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Comptroller.sol#L578

```solidity
    function getHypotheticalAccountLiquidityInternal(
        address account,
        MToken mTokenModify,
        uint redeemTokens,
        uint borrowAmount) internal view returns (Error, uint, uint) {

        AccountLiquidityLocalVars memory vars; // Holds all our calculation results
        uint oErr;

        // For each asset the account is in
        MToken[] memory assets = accountAssets[account];
        for (uint i = 0; i < assets.length; i++) {
            MToken asset = assets[i];

            // Read the balances and exchange rate from the mToken
            (oErr, vars.mTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) = asset.getAccountSnapshot(account);
            if (oErr != 0) { // semi-opaque error code, we assume NO_ERROR == 0 is invariant between upgrades
                return (Error.SNAPSHOT_ERROR, 0, 0);
            }
...
}
```

But `getAccountSnapshot` will result in error when try to calculate `exchangeRateStoredInternal` because `totalReserves` bigger than `totalCash` + `totalBorrows`.

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L357-L361


```
    function exchangeRateStoredInternal() virtual internal view returns (MathError, uint) {
        uint _totalSupply = totalSupply;
        if (_totalSupply == 0) {
            /*
             * If there are no tokens minted:
             *  exchangeRate = initialExchangeRate
             */
            return (MathError.NO_ERROR, initialExchangeRateMantissa);
        } else {
            /*
             * Otherwise:
             *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
             */
            uint totalCash = getCashPrior();
            uint cashPlusBorrowsMinusReserves;
            Exp memory exchangeRate;
            MathError mathErr;
            // @audit - this will revert if totalReservers > totalBorrow + totalCash
            (mathErr, cashPlusBorrowsMinusReserves) = addThenSubUInt(totalCash, totalBorrows, totalReserves);
            if (mathErr != MathError.NO_ERROR) {
                return (mathErr, 0);
            }

            (mathErr, exchangeRate) = getExp(cashPlusBorrowsMinusReserves, _totalSupply);
            if (mathErr != MathError.NO_ERROR) {
                return (mathErr, 0);
            }

            return (MathError.NO_ERROR, exchangeRate.mantissa);
        }
    }
```

Also this scenario will prevent `_reduceReserves` because when try to `accrueInterest`, it will also fail due to the same condition (`totalReserves` bigger than `totalCash` + `totalBorrows`) .

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L402

Now Alice can't be liquidated.

this scenario is profitable when market B has lower token value compared to market A and has low supply/borrow activity, (shortfall of Alice on market A has bigger value than the donated reserves value to market B).


## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider to restrict add reserves function, or actively monitor and remove market of token that have low/volatile value or low borrow and supply activity.








## Assessed type

Other