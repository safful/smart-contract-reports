## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Consider removing `ICore.sol`](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/56) 

# Handle

WatchPug


# Vulnerability details

Most of the interfaces defined in `ICore.sol` are unused. The only method used is `pricePerShare()` which is identical to `ICoreOracle.sol#pricePerShare()`.

Therefore, `ICore.sol` can be removed and replaced by `ICoreOracle.sol`.

