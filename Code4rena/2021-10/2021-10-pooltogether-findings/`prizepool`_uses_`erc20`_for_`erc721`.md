## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- resolved

# [`PrizePool` uses `ERC20` for `ERC721`](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/28) 

# Handle

cmichel


# Vulnerability details

The `PrizePool` defines `using SafeERC20 for IERC721;` which means the `SafeERC20.safeTransferFrom` function will be used in `awardExternalERC721`.
However, this function is just a wrapper for `contract.transferFrom` with a return-value and success check.

Thus this call actually calls `ERC721.transferFrom` instead of `ERC721.safeTransferFrom` and does not call the important `onERC721Received` check for contracts.

## Impact
ERC721s can be awarded to contracts that don't know how to handle it and they can get stuck.

## Recommended Mitigation Steps
Remove the `using SafeERC20 for IERC721;` line.


