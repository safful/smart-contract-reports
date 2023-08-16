## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [BondVault.sol: Optimizations](https://github.com/code-423n4/2021-07-spartan-findings/issues/51) 

# Handle

hickuphh3


# Vulnerability details

### Impact

- `_pool` is fetched once in `claimForMember()`, but is fetched again in its sub function `decreaseWeight()`. Since `decreaseWeight()` is solely called by `claimForMember()`, the `_pool` variable can be passed as an input to `decreaseWeight()` to avoid having to retrieve its value again.
- In `increaseWeight()` and `decreaseWeight()`, zeroing out `mapMemberPool_weight` is redundant as it is set to another value 2 lines later.

