## Tags

- bug
- 2 (Med Risk)
- judge review requested
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-09

# [Last Trove may be prevented from redeeming](https://github.com/code-423n4/2023-02-ethos-findings/issues/381) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/RedemptionHelper.sol#L128


# Vulnerability details

## Impact
In `redeemCollateral()` of RedemptionHelper.sol, the LUSD `balanceOf` the redeemer is checked against the specific collateral recorded LUSD debt (both active and defaulted). 
```solidity

function redeemCollateral(
        address _collateral,
        address _redeemer,
        uint _LUSDamount,
        address _firstRedemptionHint,
        address _upperPartialRedemptionHint,
        address _lowerPartialRedemptionHint,
        uint _partialRedemptionHintNICR,
        uint _maxIterations,
        uint _maxFeePercentage
    )
        external override
    {
        _requireCallerIsTroveManager();
        _requireValidCollateralAddress(_collateral);
        RedemptionTotals memory totals;

        _requireValidMaxFeePercentage(_maxFeePercentage);
        _requireAfterBootstrapPeriod();
        totals.price = priceFeed.fetchPrice(_collateral);
        ICollateralConfig collateralConfigCached = collateralConfig;
        totals.collDecimals = collateralConfigCached.getCollateralDecimals(_collateral);
        totals.collMCR = collateralConfigCached.getCollateralMCR(_collateral);
        _requireTCRoverMCR(_collateral, totals.price, totals.collDecimals, totals.collMCR);
        _requireAmountGreaterThanZero(_LUSDamount);
        _requireLUSDBalanceCoversRedemption(lusdToken, _redeemer, _LUSDamount);

        totals.totalLUSDSupplyAtStart = getEntireSystemDebt(_collateral);
        // Confirm redeemer's balance is less than total LUSD supply
        assert(lusdToken.balanceOf(_redeemer) <= totals.totalLUSDSupplyAtStart);
        ...
}
```
This makes sense in a single collateral system such as Liquity, but is problematic in a multi-collateral one like Reserve. Since each collateral type tracks its own debt but mints the same LUSD token, LUSD supply (and thus balance) being less than the collateral debt is no longer an invariant. This can can result in:
- Last trove may be prevented from redeeming by griefers.
- Users that deposit into multiple Trove types may be prevented from redeeming.

## Proof of Concept
### Last trove may be prevented from redeeming
Consider the cases when 
```
- There are 2 Trove types (wBTC and wETH). 
- There is 10000 total LUSD debt in the wBTC Troves.
- Stability Pool has 150 LUSD deposited i.e. full liquidity to offset debt.
- There is 100 total LUSD debt in the wETH pool.
- ETH prices crash and all Troves get liquidated except the last one.
```
A griefer can front-run the last Trove from redeeming by sending the user weth with the amount `entireSystemDebt` + 1.

In a similar case as above, any users that may borrow from multiple Troves types such that their LUSD balance is greater than the total collateral debt will be prevented from redeeming. However, this is not as problematic because they can just send their excess tokens out.

## Tools Used
Manual Review

## Recommended Mitigation Steps
- Consider removing this check as the invariant no longer applies