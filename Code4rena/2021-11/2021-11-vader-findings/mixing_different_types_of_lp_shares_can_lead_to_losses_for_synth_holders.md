## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- VaderPoolV2
- BasePoolV2

# [Mixing different types of LP shares can lead to losses for Synth holders](https://github.com/code-423n4/2021-11-vader-findings/issues/257) 

# Handle

hyh


# Vulnerability details


## Impact

Users that mint Synths do not get pool shares, so exiting of normal LP can lead to their losses as no funds can be left for retrieval.

## Proof of Concept

3 types of mint/burn: NFT, Fungible and Synths. Synths are most vilnerable as they do not have share: LP own the pool, so Synth's funds are lost in scenarios similar to:
1. LP deposit both sides to a pool
2. Synth deposit and mint a Synth
3. LP withdraws all as she owns all the pool liquidity, even when provided only part of it
4. Synth can't withdraw as no assets left

burn NFT LP:
https://github.com/code-423n4/2021-11-vader/blob/main/contracts/dex-v2/pool/BasePoolV2.sol#L270

burn fungible LP:
https://github.com/code-423n4/2021-11-vader/blob/main/contracts/dex-v2/pool/VaderPoolV2.sol#L374

## Recommended Mitigation Steps

Take into account liquidity that was provided by Synth minting.

