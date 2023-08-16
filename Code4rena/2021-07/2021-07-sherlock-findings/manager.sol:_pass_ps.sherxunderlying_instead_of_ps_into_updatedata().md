## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Manager.sol: Pass ps.sherXUnderlying instead of ps into updateData()](https://github.com/code-423n4/2021-07-sherlock-findings/issues/40) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The only value from poolStorage in `updateData()` is the `sherXUnderlying` value. It is cheaper to pass this `uint256` variable instead of the storage variable itself.

### Recommended Mitigation Steps

Change `PoolStorage.Base storage ps` to `uint256 sherXUnderlying`, saves about ~160 gas.

