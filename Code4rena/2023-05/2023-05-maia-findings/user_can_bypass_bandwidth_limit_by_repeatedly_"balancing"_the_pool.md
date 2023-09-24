## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-20

# [User can bypass bandwidth limit by repeatedly "balancing" the pool](https://github.com/code-423n4/2023-05-maia-findings/issues/392) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesPool.sol#L539-L671


# Vulnerability details

## Impact
The goal with bandwidths is to have a maximum that can be withdrawn (swapped) from a pool, so in case a specific chain (or token from a chain) is exploited, then it only can partially affect these pools. However, the maximum limit can be bypassed by repeatedly "balancing" the pool to increase bandwidth for the exploited chain.

## Introducing "Balancing": A Technique for Redistributing Bandwidth
During `ulyssesAddLP` or `ulyssesAddLP`, liquidity is first distributed or taken proportionally to `diff` (if any exists), then distributed or taken proportionally to `weight`. Suppose integer `t` is far smaller than `diff` (since the action itself can also change `diff`), after repeatedly adding t LP, removing t lp, adding t LP, removing t Lp ...... the pool will finally reach another stable state where the ratio of `diff` to `weight` is a constant among destinations. This implies that the `currentBandwidth` will be proportional to `weight`.

## Proof of Concept
Suppose Avalanche is down. Unluckily, Alice holds 100 ava-hETH. She wants to swap ava-hETH for bnb-hETH.

Let's take a look at bnb-hETH pool. Suppose weights are mainnet:4, Avalanche:3, Linea:2. Total supply is 90. Target bandwidths are mainnet:40, Avalanche:30, Linea:20. Current bandwidths are mainnet:30, Avalanche:2(few left), Linea:22.

Ideally Alice should only be able to swap for 2 bnb-hETH. However, she swaps for 0.1 bnb-hETH first. Then she uses the 0.1 bnb-hETH to "balance" the pool (as mentioned above). Current bandwidths will become mainnet:24, Avalanche:18, Linea:12. Then Alice swaps for 14 bnb-hETH and "balance" the pool again. By repeating the process, she can acquire nearly all of the available liquidity in pool and LP loss will be unbounded.

## Tools Used
Manual Review

## Recommended Mitigation Steps
1. During `ulyssesAddLP` or `ulyssesAddLP`, always distribute or take liquidity proportionally to weight.
2. When swapping A for B, reduce bandwidth of A in B pool (as is currently done) while add bandwidth of B in A pool (instead of distributing them among all bandwidths).


## Assessed type

Context