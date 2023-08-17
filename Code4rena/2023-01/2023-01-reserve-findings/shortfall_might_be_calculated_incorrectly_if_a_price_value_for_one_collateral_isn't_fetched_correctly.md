## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-20

# [Shortfall might be calculated incorrectly if a price value for one collateral isn't fetched correctly](https://github.com/code-423n4/2023-01-reserve-findings/issues/200) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/mixins/RecollateralizationLib.sol#L449


# Vulnerability details

## Impact
Function `price()` of an asset doesn't revert. It [returns values](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/Asset.sol#L117) `(0, FIX_MAX)` for `low, high` values of price in case there's a problem with fetching it. Code that calls `price()` is able to validate returned values to detect that returned price is incorrect.

Inside function `collateralShortfall()` of `RecollateralizationLibP1` [collateral price isn't checked for correctness](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/mixins/RecollateralizationLib.sol#L449). As a result incorrect value of `shortfall` might be [calculated](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/mixins/RecollateralizationLib.sol#L452) if there are difficulties to fetch a price for one of the collaterals. 

## Proof of Concept
* https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/mixins/RecollateralizationLib.sol#L449

## Recommended Mitigation Steps
Check that price is correctly fetched for a collateral.