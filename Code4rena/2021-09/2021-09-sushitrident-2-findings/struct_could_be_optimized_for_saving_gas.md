## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Struct could be optimized for saving gas](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/57) 

# Handle

WatchPug


# Vulnerability details

Members of structs should be grouped into bunches of 32 bytes for saving gas.

For example:

https://github.com/sushiswap/trident/blob/c405f3402a1ed336244053f8186742d2da5975e9/contracts/pool/concentrated/ConcentratedLiquidityPoolManager.sol#L15-L23

`ConcentratedLiquidityPoolManager.sol#Incentive` `rewardsUnclaimed` and `secondsClaimed` can be moved to the bottom to optimize for Variable Packing.

