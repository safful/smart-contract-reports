## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [NFTXMarketplaceZap: Add rescue() function](https://github.com/code-423n4/2021-12-nftx-findings/issues/226) 

# Handle

GreyArt


# Vulnerability details

## Impact

A `rescue()` function exists for the StakingZap contract to help retrieve any accidental fund transfer to it. It would be beneficial to have this function exist in the MarketplaceZap contract too.

## Recommended Mitigation Steps

Include the `rescue()` function.

```jsx
function rescue(address token) external onlyOwner {
	IERC20Upgradeable(token).transfer(msg.sender, IERC20Upgradeable(token).balanceOf(address(this)));
}
```

