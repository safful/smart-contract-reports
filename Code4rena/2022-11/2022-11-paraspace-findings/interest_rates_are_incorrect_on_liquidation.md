## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-03

# [Interest rates are incorrect on Liquidation](https://github.com/code-423n4/2022-11-paraspace-findings/issues/173) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/main/paraspace-core/contracts/protocol/libraries/logic/LiquidationLogic.sol#L531


# Vulnerability details

## Impact
The debt tokens are being transferred before calculating the interest rates. But the interest rate calculation function assumes that debt token has not yet been sent thus the outcome `currentLiquidityRate` will be incorrect

## Proof of Concept
1. Liquidator L1 calls [`executeLiquidateERC20`](https://github.com/code-423n4/2022-11-paraspace/blob/main/paraspace-core/contracts/protocol/libraries/logic/LiquidationLogic.sol#L161) for a position whose health factor <1

```
function executeLiquidateERC20(
        mapping(address => DataTypes.ReserveData) storage reservesData,
        mapping(uint256 => address) storage reservesList,
        mapping(address => DataTypes.UserConfigurationMap) storage usersConfig,
        DataTypes.ExecuteLiquidateParams memory params
    ) external returns (uint256) {

...
 _burnDebtTokens(liquidationAssetReserve, params, vars);
...
}
```

2. This internally calls [`_burnDebtTokens`](https://github.com/code-423n4/2022-11-paraspace/blob/main/paraspace-core/contracts/protocol/libraries/logic/LiquidationLogic.sol#L523)

```
    function _burnDebtTokens(
        DataTypes.ReserveData storage liquidationAssetReserve,
        DataTypes.ExecuteLiquidateParams memory params,
        ExecuteLiquidateLocalVars memory vars
    ) internal {
       ...

        // Transfers the debt asset being repaid to the xToken, where the liquidity is kept
        IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
...
        // Update borrow & supply rate
        liquidationAssetReserve.updateInterestRates(
            vars.liquidationAssetReserveCache,
            params.liquidationAsset,
            vars.actualLiquidationAmount,
            0
        );
    }
```

3. Basically first it transfers the debt asset to xToken using below. This increases the balance of xTokenAddress for liquidationAsset

```
IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
```

4. Now `updateInterestRates` function is called on ReserveLogic.sol#L169

```
function updateInterestRates(
        DataTypes.ReserveData storage reserve,
        DataTypes.ReserveCache memory reserveCache,
        address reserveAddress,
        uint256 liquidityAdded,
        uint256 liquidityTaken
    ) internal {
...
(
            vars.nextLiquidityRate,
            vars.nextVariableRate
        ) = IReserveInterestRateStrategy(reserve.interestRateStrategyAddress)
            .calculateInterestRates(
                DataTypes.CalculateInterestRatesParams({
                    liquidityAdded: liquidityAdded,
                    liquidityTaken: liquidityTaken,
                    totalVariableDebt: vars.totalVariableDebt,
                    reserveFactor: reserveCache.reserveFactor,
                    reserve: reserveAddress,
                    xToken: reserveCache.xTokenAddress
                })
            );
...
}
```

5. Finally call to `calculateInterestRates` function on DefaultReserveInterestRateStrategy#L127 contract is made which calculates the interest rate

```
function calculateInterestRates(
        DataTypes.CalculateInterestRatesParams calldata params
    ) external view override returns (uint256, uint256) {
...
if (vars.totalDebt != 0) {
            vars.availableLiquidity =
                IToken(params.reserve).balanceOf(params.xToken) +
                params.liquidityAdded -
                params.liquidityTaken;

            vars.availableLiquidityPlusDebt =
                vars.availableLiquidity +
                vars.totalDebt;
            vars.borrowUsageRatio = vars.totalDebt.rayDiv(
                vars.availableLiquidityPlusDebt
            );
            vars.supplyUsageRatio = vars.totalDebt.rayDiv(
                vars.availableLiquidityPlusDebt
            );
        }
...
vars.currentLiquidityRate = vars
            .currentVariableBorrowRate
            .rayMul(vars.supplyUsageRatio)
            .percentMul(
                PercentageMath.PERCENTAGE_FACTOR - params.reserveFactor
            );

        return (vars.currentLiquidityRate, vars.currentVariableBorrowRate);
}
```

6. As we can see in above code, `vars.availableLiquidity` is calculated as `IToken(params.reserve).balanceOf(params.xToken) +params.liquidityAdded - params.liquidityTaken`

7. But the problem is that debt token is already transferred to `xToken` which means `xToken` already consist of `params.liquidityAdded`. Hence the calculation ultimately becomes `(xTokenBeforeBalance+params.liquidityAdded) +params.liquidityAdded - params.liquidityTaken`

8. This is incorrect and would lead to higher `vars.availableLiquidity` which ultimately impacts the `currentLiquidityRate`

## Recommended Mitigation Steps
Transfer the debt asset post interest calculation

```
function _burnDebtTokens(
        DataTypes.ReserveData storage liquidationAssetReserve,
        DataTypes.ExecuteLiquidateParams memory params,
        ExecuteLiquidateLocalVars memory vars
    ) internal {
IPToken(vars.liquidationAssetReserveCache.xTokenAddress)
            .handleRepayment(params.liquidator, vars.actualLiquidationAmount);
        // Burn borrower's debt token
        vars
            .liquidationAssetReserveCache
            .nextScaledVariableDebt = IVariableDebtToken(
            vars.liquidationAssetReserveCache.variableDebtTokenAddress
        ).burn(
                params.borrower,
                vars.actualLiquidationAmount,
                vars.liquidationAssetReserveCache.nextVariableBorrowIndex
            );

liquidationAssetReserve.updateInterestRates(
            vars.liquidationAssetReserveCache,
            params.liquidationAsset,
            vars.actualLiquidationAmount,
            0
        );
IERC20(params.liquidationAsset).safeTransferFrom(
            vars.payer,
            vars.liquidationAssetReserveCache.xTokenAddress,
            vars.actualLiquidationAmount
        );
...
...
}
```