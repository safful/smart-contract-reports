## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- M-01

# [Potential risk of using `swappedAmount` in case of swap error](https://github.com/code-423n4/2023-06-canto-findings/issues/71) 

# Lines of code

https://github.com/code-423n4/2023-06-canto/blob/main/Canto/x/onboarding/keeper/ibc_callbacks.go#L93-L96
https://github.com/code-423n4/2023-06-canto/blob/a4ff2fd2e67e77e36528fad99f9d88149a5e8532/Canto/x/onboarding/keeper/ibc_callbacks.go#L124


# Vulnerability details

## Impact
In case the swap operation failed, the module should continue as is with the erc20 conversion and finish the IBC transfer. This is the relevant part of the code that swallows the error:
```
swappedAmount, err = k.coinswapKeeper.TradeInputForExactOutput(ctx, coinswaptypes.Input{Coin: transferredCoin, Address: recipient.String()}, coinswaptypes.Output{Coin: swapCoins, Address: recipient.String()})
if err != nil {
    logger.Error("failed to swap coins", "error", err)
} 
```
Notice that in case of an error, `swappedAmount` will still be written to. 
Later on in the code, it is used to calculate the conversion amount:
```
convertCoin := sdk.NewCoin(transferredCoin.Denom, transferredCoin.Amount.Sub(swappedAmount))
```

The `swappedAmount` is trusted to have a zero value in this case. While this is currently true in the existing code, variables returned in error states should not be trusted and should be overwritten.
Currently all error states return `sdk.ZeroInt()` unless the swap was executed correctly, but it might change in  a future PR.

## Proof of Concept
1. Run this patch, it will cause TradeInputForExactOutput to always error with a swappedAmount > 0 .
```
diff --git a/Canto/x/coinswap/keeper/swap.go b/Canto/x/coinswap/keeper/swap.go
index 67e04ef..a331bcf 100644
--- a/Canto/x/coinswap/keeper/swap.go
+++ b/Canto/x/coinswap/keeper/swap.go
@@ -2,6 +2,7 @@ package keeper
 
 import (
        "fmt"
+
        sdk "github.com/cosmos/cosmos-sdk/types"
        sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
 
@@ -160,51 +161,8 @@ Buy exact amount of a token by specifying the max amount of another token, one o
 @param receipt : address of the receiver
 @return : actual amount of the token to be paid
 */
-func (k Keeper) TradeInputForExactOutput(ctx sdk.Context, input types.Input, output types.Output) (sdk.Int, error) {
-       soldTokenAmt, err := k.calculateWithExactOutput(ctx, output.Coin, input.Coin.Denom)
-       if err != nil {
-               return sdk.ZeroInt(), err
-       }
-
-       // assert that the calculated amount is less than the
-       // max amount the buyer is willing to pay.
-       if soldTokenAmt.GT(input.Coin.Amount) {
-               return sdk.ZeroInt(), sdkerrors.Wrap(types.ErrConstraintNotMet, fmt.Sprintf("insufficient amount of %s, user expected: %s, actual: %s", input.Coin.Denom, input.Coin.Amount.String(), soldTokenAmt.String()))
-       }
-       soldToken := sdk.NewCoin(input.Coin.Denom, soldTokenAmt)
-
-       inputAddress, err := sdk.AccAddressFromBech32(input.Address)
-       if err != nil {
-               return sdk.ZeroInt(), err
-       }
-       outputAddress, err := sdk.AccAddressFromBech32(output.Address)
-       if err != nil {
-               return sdk.ZeroInt(), err
-       }
-
-       standardDenom := k.GetStandardDenom(ctx)
-       var quoteCoinToSwap sdk.Coin
-
-       if soldToken.Denom != standardDenom {
-               quoteCoinToSwap = soldToken
-       } else {
-               quoteCoinToSwap = output.Coin
-       }
-
-       maxSwapAmount, err := k.GetMaximumSwapAmount(ctx, quoteCoinToSwap.Denom)
-       if err != nil {
-               return sdk.ZeroInt(), err
-       }
-
-       if quoteCoinToSwap.Amount.GT(maxSwapAmount.Amount) {
-               return sdk.ZeroInt(), sdkerrors.Wrap(types.ErrConstraintNotMet, fmt.Sprintf("expected swap amount %s%s exceeding swap amount limit %s%s", quoteCoinToSwap.Amount.String(), quoteCoinToSwap.Denom, maxSwapAmount.Amount.String(), maxSwapAmount.Denom))
-       }
-
-       if err := k.swapCoins(ctx, inputAddress, outputAddress, soldToken, output.Coin); err != nil {
-               return sdk.ZeroInt(), err
-       }
-
-       return soldTokenAmt, nil
+func (k Keeper) TradeInputForExactOutput(_ sdk.Context, _ types.Input, _ types.Output) (sdk.Int, error) {
+       return sdk.NewIntFromUint64(10000), fmt.Errorf("swap error")
 }
```

2. Add this test to `x/onboarding/keeper/ibc_callbacks_test.go` :
```
{
    "swap fails with swappedAmount / convert remaining ibc token - some vouchers are not converted",
    func() {
        transferAmount = sdk.NewIntWithDecimal(25, 6)
        transfer := transfertypes.NewFungibleTokenPacketData(denom, transferAmount.String(), secpAddrCosmos, ethsecpAddrcanto)
        bz := transfertypes.ModuleCdc.MustMarshalJSON(&transfer)
        packet = channeltypes.NewPacket(bz, 100, transfertypes.PortID, sourceChannel, transfertypes.PortID, cantoChannel, timeoutHeight, 0)
    },
    true,
    sdk.NewCoins(sdk.NewCoin("acanto", sdk.NewIntWithDecimal(3, 18))),
    sdk.NewCoin("acanto", sdk.NewIntWithDecimal(3, 18)),
    sdk.NewCoin(uusdcIbcdenom, sdk.NewIntFromUint64(10000)),
    sdk.NewInt(24990000),
},
```

The test will still fail because of another unimportant check, but the important check will pass - the address will have `sent-swappedAmount` vouchers converted, and the rest will be kept.
It means swappedAmount was used even though the swap function failed.

## Tools Used
IDE.

## Recommended Mitigation Steps
Zero the `swappedAmount` variable in the error case:
```
swappedAmount = sdk.ZeroInt()
```


## Assessed type

Other