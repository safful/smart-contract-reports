## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [Non-ideal rETH/WETH pool used pays unnecessary fees](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/673) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L101


# Vulnerability details

## Impact

rETH is acquired using the Uniswap rETH/WETH pool. This solution has higher fees and lower liquidity than alternatives, which results in more lost user value than other solutions.

The Uniswap rETH/WETH pool that is used in Reth.sol to make swaps has a liquidity of $5 million. In comparison, the Balancer rETH/WETH pool has a liquidity of $80 million. Even the Curve rETH/WETH pool has a liquidity of $8 million. The greater liquidity should normally offer lower slippage to users. In addition, the fees to swap with the Balancer pool are only 0.04% compared to Uniswap's 0.05%. Even the Curve pool offers a lower fee than Uniswap with just a 0.037% fee. [This Dune Analytics dashboard](https://dune.com/drworm/rocketpool) shows that Balancer is where the majority of rETH swaps happen by volume.

One solution to finding the best swap path for rETH is to use RocketPool's [RocketSwapRouter.sol contract `swapTo()` function](https://etherscan.io/address/0x16d5a408e807db8ef7c578279beeee6b228f1c1c#code#F19#L64). When users visit the RocketPool frontend to swap ETH for rETH, this is the function that RocketPool calls for the user. RocketSwapRouter.sol automatically determines the best way to split the swap between Balancer and Uniswap pools.

## Proof of Concept

Pools that can be used for rETH/WETH swapping:
- [Uniswap rETH/WETH pool](https://etherscan.io/address/0xa4e0faA58465A2D369aa21B3e42d43374c6F9613): $5 million in liquidity
- [Balancer rETH/WETH pool](https://app.balancer.fi/#/ethereum/pool/0x1e19cf2d73a72ef1332c882f20534b6519be0276000200000000000000000112)
- [Curve Finance rETH/ETH pool](https://curve.fi/#/ethereum/pools/factory-crypto-210/deposit): $8 million in liquidity

[Line where Reth.sol swaps WETH for rETH](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L101) with the Uniswap rETH/WETH pool.

## Tools Used

Etherscan, Dune Analytics

## Recommended Mitigation Steps

The best solution is to use the same flow as RocketPool's frontend UI and to call `swapTo()` in [RocketSwapRouter.sol](https://etherscan.io/address/0x16d5a408e807db8ef7c578279beeee6b228f1c1c#code#F19#L64). An alternative is to modify Reth.sol to use the Balancer rETH/ETH pool for swapping instead of Uniswap's rETH/WETH pool to better conserve user value by reducing swap fees and reducing slippage costs.