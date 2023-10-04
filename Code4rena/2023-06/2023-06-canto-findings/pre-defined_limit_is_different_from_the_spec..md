## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- upgraded by judge
- H-01

# [Pre-defined limit is different from the spec.](https://github.com/code-423n4/2023-06-canto-findings/issues/36) 

# Lines of code

https://github.com/code-423n4/2023-06-canto/blob/main/Canto/x/coinswap/keeper/swap.go#L212
https://github.com/code-423n4/2023-06-canto/blob/main/Canto/x/coinswap/types/params.go#L34


# Vulnerability details

## Impact

In the spec, the pre-defined limit of ETH is 0.01 ETHs. But the actual limit in the code is not 0.01 ETH which could result in misleading.

## Proof of Concept

In the spec, it said that the pre-defined limit of ETH is 0.01 ETHs
https://github.com/code-423n4/2023-06-canto/blob/main/README.md#swap
> For risk management purposes, a swap will fail if the input coin amount exceeds a pre-defined limit (10 USDC, 10 USDT, 0.01 ETH) or if the swap amount limit is not defined.

But in `x/coinswap/types/params.go`, the actual limit of ETH is 1*10e17 which is 0.1 ETH
```solidity
// Parameter store keys
var (
	KeyFee                    = []byte("Fee")                    // fee key
	KeyPoolCreationFee        = []byte("PoolCreationFee")        // fee key
	KeyTaxRate                = []byte("TaxRate")                // fee key
	KeyStandardDenom          = []byte("StandardDenom")          // standard token denom key
	KeyMaxStandardCoinPerPool = []byte("MaxStandardCoinPerPool") // max standard coin amount per pool
	KeyMaxSwapAmount          = []byte("MaxSwapAmount")          // whitelisted denoms

	DefaultFee                    = sdk.NewDecWithPrec(0, 0)
	DefaultPoolCreationFee        = sdk.NewInt64Coin(sdk.DefaultBondDenom, 0)
	DefaultTaxRate                = sdk.NewDecWithPrec(0, 0)
	DefaultMaxStandardCoinPerPool = sdk.NewIntWithDecimal(10000, 18)
	DefaultMaxSwapAmount          = sdk.NewCoins(
		sdk.NewCoin(UsdcIBCDenom, sdk.NewIntWithDecimal(10, 6)),
		sdk.NewCoin(UsdtIBCDenom, sdk.NewIntWithDecimal(10, 6)),
		sdk.NewCoin(EthIBCDenom, sdk.NewIntWithDecimal(1, 17)),
	)
)
```

The limit is used in `swap.GetMaximumSwapAmount`. Wrong could harm the risk management.
https://github.com/code-423n4/2023-06-canto/blob/main/Canto/x/coinswap/keeper/swap.go#L212
```solidity
func (k Keeper) GetMaximumSwapAmount(ctx sdk.Context, denom string) (sdk.Coin, error) {
	params := k.GetParams(ctx)
	for _, coin := range params.MaxSwapAmount {
		if coin.Denom == denom {
			return coin, nil
		}
	}
	return sdk.Coin{}, sdkerrors.Wrap(types.ErrInvalidDenom, fmt.Sprintf("invalid denom: %s, denom is not whitelisted", denom))
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

0.01 ETH should be `sdk.NewIntWithDecimal(1, 16)`



## Assessed type

Error