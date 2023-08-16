## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [`HybridPool`'s `flashSwap` sends entire fee to `barFeeTo`](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/99) 

# Handle

cmichel


# Vulnerability details

The `HybridPool.flashSwap` function sends the entire trade fees `fee` to the `barFeeTo`.
It should only send `barFee * fee` to the `barFeeTo` address.

## Impact
LPs are not getting paid at all when this function is used.
There is no incentive to provide liquidity.

## Recommended Mitigation Steps
The `flashSwap` function should use the same fee mechanism as `swap` and only send `barFee * fee / MAX_FEE` to the `barFeeTo`. See `_handleFee` function.


