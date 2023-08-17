## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-18

# [Bad debt will likely incur when multiple NFTs are liquidated.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/479) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/GenericLogic.sol#L394


# Vulnerability details

## Description

\_getUserBalanceForERC721() in GenericLogic gets the value of a user's specific ERC721 xToken. It is later used for determining the account's health factor. In case `isAtomicPrice` is false such as in ape NTokens, price is calculated using:

```
    uint256 assetPrice = _getAssetPrice(
        params.oracle,
        vars.currentReserveAddress
    );
    totalValue =
        ICollateralizableERC721(vars.xTokenAddress)
            .collateralizedBalanceOf(params.user) *
        assetPrice;
```

It is the number of apes multiplied by the floor price returns from \_getAssetPrice. The issue is that this calculation does not account for slippage, and is unrealistic. If user's account is liquidated, it is very unlikely that releasing several multiples of precious NFTs will not bring the price down in some significant way. 

By performing simple multiplication of NFT count and NFT price, protocol is introducing major bad debt risks and is not as conservative as it aims to be. Collateral value must take slippage risks into account.

## Impact

Bad debt will likely incur when multiple NFTs are liquidated.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Change the calculation to account for slippage when NFT balance is above some threshold.