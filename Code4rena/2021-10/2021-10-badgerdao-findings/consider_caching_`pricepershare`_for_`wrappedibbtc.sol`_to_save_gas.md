## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Consider caching `pricePerShare` for `WrappedIbbtc.sol` to save gas](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/55) 

# Handle

WatchPug


# Vulnerability details

The current implementation of `WrappedIbbtc.sol` will do an external call `oracle.pricePerShare()` every time `pricePerShare` is used, it can be gas consuming considering that the basic features include: `balanceOf()`, `transfer()`, `transferFrom()` will be used very often.

### Recommendation

Consider caching `pricePerShare` in storage.

