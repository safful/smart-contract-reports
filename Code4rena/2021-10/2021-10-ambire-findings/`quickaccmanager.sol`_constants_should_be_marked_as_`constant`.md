## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [`QuickAccManager.sol` Constants should be marked as `constant`](https://github.com/code-423n4/2021-10-ambire-findings/issues/31) 

# Handle

WatchPug


# Vulnerability details

https://github.com/code-423n4/2021-10-ambire/blob/bc01af4df3f70d1629c4e22a72c19e6a814db70d/contracts/wallet/QuickAccManager.sol#L128-L128

The variables `TRANSFER_TYPEHASH`, `TXNS_TYPEHASH`, `BUNDLE_TYPEHASH` are named in all caps, which implies that they are constants. However, they are not being marked as `constant`. Mark them as `constant` can also help save some gas.

