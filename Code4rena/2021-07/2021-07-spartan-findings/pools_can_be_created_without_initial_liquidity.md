## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- disagree with severity

# [Pools can be created without initial liquidity](https://github.com/code-423n4/2021-07-spartan-findings/issues/151) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

The protocol differentiates between public pool creations and private ones (starting without liquidity).
However, this is not effective as anyone can just flashloan the required initial pool liquidity, call `PoolFactory.createPoolADD`, receive the LP tokens in `addForMember` and withdraw liquidity again.

## Recommended Mitigation Steps
Consider burning some initial LP tokens or taking a pool creation fee instead.


