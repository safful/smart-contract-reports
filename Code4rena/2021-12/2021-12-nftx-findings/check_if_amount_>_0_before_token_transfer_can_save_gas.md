## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Check if amount > 0 before token transfer can save gas](https://github.com/code-423n4/2021-12-nftx-findings/issues/165) 

# Handle

WatchPug


# Vulnerability details

https://github.com/code-423n4/2021-12-nftx/blob/194073f750b7e2c9a886ece34b6382b4f1355f36/nftx-protocol-v2/contracts/solidity/NFTXMarketplaceZap.sol#L247-L248

```solidity=247
uint256 remaining = WETH.balanceOf(address(this));
WETH.transfer(to, remaining);
```

Since `WETH.balanceOf(address(this))` can to be `0`. Checking `if (remaining > 0)` before the transfer can potentially save an external call and the unnecessary gas cost of a 0 token transfer.

