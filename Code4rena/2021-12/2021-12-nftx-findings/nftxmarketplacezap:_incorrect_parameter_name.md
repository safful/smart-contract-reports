## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- resolved
- sponsor confirmed

# [NFTXMarketplaceZap: incorrect parameter name](https://github.com/code-423n4/2021-12-nftx-findings/issues/228) 

# Handle

GreyArt


# Vulnerability details

## Impact

In the function `_sellVaultTokenETH`, the parameter `minWethOut` should be `minEthOut`

## Recommended Mitigation Steps

Replace `minWethOut` with `minEthOut`

