## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [hybrid pool uses wrong `non_optimal_mint_fee`](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/31) 

# Handle

broccoli


# Vulnerability details

## Impact

When an lp provider deposits an imbalance amount of token, a swap fee is applied. HybridPool uses the same `_nonOptimalMintFee` as `constantProductPool`; however, since two pools use different AMM curve, the ideal balance is not the same.  ref: [StableSwap3Pool.vy#L322-L337](https://github.com/curvefi/curve-contract/blob/master/contracts/pools/3pool/StableSwap3Pool.vy#L322-L337)

Stable swap Pools are designed for 1B+ TVL. Any issue related to pricing/fee is serious. I consider this is a high-risk issue

## Proof of Concept

[StableSwap3Pool.vy#L322-L337](https://github.com/curvefi/curve-contract/blob/master/contracts/pools/3pool/StableSwap3Pool.vy#L322-L337)

[HybridPool.sol#L425-L441](https://github.com/sushiswap/trident/blob/9130b10efaf9c653d74dc7a65bde788ec4b354b5/contracts/pool/HybridPool.sol#L425-L441)

## Tools Used

None

## Recommended Mitigation Steps
Calculate the swapping fee based on the stable swap curve.
Please refer to [StableSwap3Pool.vy#L322-L337](https://github.com/curvefi/curve-contract/blob/master/contracts/pools/3pool/StableSwap3Pool.vy#L322-L337).

