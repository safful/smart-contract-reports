## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [NFTXStakingZap: Unused xTokenMinted variable ](https://github.com/code-423n4/2021-12-nftx-findings/issues/217) 

# Handle

GreyArt


# Vulnerability details

## Impact

`xTokensMinted` is assigned in `provideInventory721()` and `provideInventory1155()`, but is unused.

## Recommended Mitigation Steps

Remove the local variable `xTokensMinted`.

