## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- low quality report
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [BORROWERS CAN AVOID LIQUIDATIONS, IF ERC777 TOKEN IS CONFIGURED AS AN `emissionToken` ](https://github.com/code-423n4/2023-07-moonwell-findings/issues/343) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MErc20.sol#L139-L142
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L1002
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L1235-L1239


# Vulnerability details

## Impact

If a borrower is `undercollateralized` then he can be liquidated by a liquidator by calling the `MErc20.liquidateBorrow` function. `liquidateBorrow` function calls the `MToken.liquidateBorrowFresh` in its execution process. Inside the `liquidateBorrowFresh` function the `MToken.repayBorrowFresh` is called which verifies whether repayment of borrowed amount is allowed by calling the `Comptroller.repayBorrowAllowed` function. The `repayBorrowAllowed` function updates the `borrower eligible rewards` by calling the `MultiRewardDistributor.updateMarketBorrowIndexAndDisburseBorrowerRewards`.

The `MultiRewardDistributor.updateMarketBorrowIndexAndDisburseBorrowerRewards` function calls the `disburseBorrowerRewardsInternal` to ditribute the multi rewards to the `borrower`.
The `disburseBorrowerRewardsInternal` calls the `sendReward` function if the `_sendTokens` flag is set to `true`.

`sendReward` function is called for each `emissionConfig.config.emissionToken` token of the `MarketEmissionConfig[]` array of the given `_mToken` market.

`sendReward` function transafers the rewards to the `borrower` using the `safeTransfer` function as shown below:

            token.safeTransfer(_user, _amount);

Problem here is that `emissionToken` can be `ERC777` token (which is backward compatible with ERC20) thus allowing a `malicious borrower` (the `recipient contract` of rewards in this case) to implement `tokensReceived` hook in its contract and revert the transaction inside the hook. This will revert the entire `liquidation` transaction. Hence the `undercollateralized borrower` can avoid the liquidation thus putting both depositors and protocol in danger.

## Proof of Concept

```solidity
    function liquidateBorrow(address borrower, uint repayAmount, MTokenInterface mTokenCollateral) override external returns (uint) {
        (uint err,) = liquidateBorrowInternal(borrower, repayAmount, mTokenCollateral);
        return err;
    }
```

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MErc20.sol#L139-L142

```solidity
        (uint repayBorrowError, uint actualRepayAmount) = repayBorrowFresh(liquidator, borrower, repayAmount);
```

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L1002

```solidity
        if (_amount > 0 && _amount <= currentTokenHoldings) {
            // Ensure we use SafeERC20 to revert even if the reward token isn't ERC20 compliant
            token.safeTransfer(_user, _amount);
            return 0;
        } else {
```

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L1235-L1239

## Tools Used
Manual Review and VSCode

## Recommended Mitigation Steps
Hence it is recommended to disallow any ERC777 tokens being configured as `emissionToken` in any `MToken` market and only allow ERC20 tokens as `emissionTokens`.


## Assessed type

Other