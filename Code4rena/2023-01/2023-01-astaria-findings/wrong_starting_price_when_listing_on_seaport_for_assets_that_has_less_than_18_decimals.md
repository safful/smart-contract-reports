## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-15

# [Wrong starting price when listing on Seaport for assets that has less than 18 decimals](https://github.com/code-423n4/2023-01-astaria-findings/issues/235) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L639-L647


# Vulnerability details

## Impact
According to Astaria's docs:
https://docs.astaria.xyz/docs/protocol-mechanics/loanterms
> Liquidation initial ask: Should the NFT go into liquidation, the initial price of the auction will be set to this value. Note that this set as a starting point for a dutch auction, and the price will decrease over the liquidation period. This figure is should also be specified in 10^18 format.

The liquidation initial ask is specififed in 18 decimals. this is then used as a starting price when the NFT goes under auction on OpenSea. However, if the  asset has less than 18 decimals, then the starting price goes wrong to Seaport. 

This results in listing the NFT with too high price that makes it unlikely to be sold.


## Proof of Concept

The starting price is set to the liquidation initial ask:
```sh
    listedOrder = s.COLLATERAL_TOKEN.auctionVault(
      ICollateralToken.AuctionVaultParams({
        settlementToken: stack[position].lien.token,
        collateralId: stack[position].lien.collateralId,
        maxDuration: auctionWindowMax,
        startingPrice: stack[0].lien.details.liquidationInitialAsk,
        endingPrice: 1_000 wei
      })
    );
```
https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L639-L647

Let's assume the asset is USDC which has 6 decimals:
1. Stratigist signs a strategy with liquidationInitialAsk **1000e18**.
2. Following the docs, this means the starting price is supposed to be **1000** USDC
3. The NFT is being liquidated.
4. 1000e18 is passed to Seaport along with asset USDC.
5. Seaport lists the NFT, and the price will be too high as1000e18 will be **1000000000000000** USDC


## Tools Used
Manual analysis

## Recommended Mitigation Steps

1. Either fetch the asset's decimals on-chain or add it as a part of the strategy.
2. Convert liquidationInitialAsk to the asset's decimals before passing it as a starting price.  

