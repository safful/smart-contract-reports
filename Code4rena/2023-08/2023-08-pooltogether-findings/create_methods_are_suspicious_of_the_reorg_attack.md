## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- M-09

# [create methods are suspicious of the reorg attack](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/31) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBoosterFactory.sol#L29


# Vulnerability details

## Impact

The createVaultBooster() function deploys a new VaultBooster contract using the ```create```, where the address derivation depends only on the VaultBoosterFactory nonce.

Re-orgs can happen in all EVM chains and as confirmed the contracts will be deployed on most EVM compatible L2s including Arbitrum, etc. It is also planned to be deployed on ZKSync in future. In ethereum, where this is deployed, Re-orgs has already been happened. For more info, [check here](https://decrypt.co/101390/ethereum-beacon-chain-blockchain-reorg)

This issue will increase as some of the chains like Arbitrum and Polygon are suspicious of the reorg attacks.

Polygon re-org reference: [click here](https://protos.com/polygon-hit-by-157-block-reorg-despite-hard-fork-to-reduce-reorgs/) This one happened this year in February, 2023.

Polygon blocks forked: [check here](https://polygonscan.com/blocks_forked)

The issue would happen when users rely on the address derivation in advance or try to deploy the position clone with the same address on different EVM chains, any funds sent to the ```new``` contract could potentially be withdrawn by anyone else. All in all, it could lead to the theft of user funds.


```Solidity
File: src/VaultBoosterFactory.sol

    function createVaultBooster(PrizePool _prizePool, address _vault, address _owner) external returns (VaultBooster) {
>>        VaultBooster booster = new VaultBooster(_prizePool, _vault, _owner);

        emit CreatedVaultBooster(booster, _prizePool, _vault, _owner);

        return booster;
    }
```

Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation.

**Attack Scenario**
Imagine that Alice deploys a new VaultBooster, and then sends funds to it. Bob sees that the network block reorg happens and calls createVaultBooster. Thus, it creates VaultBooster with an address to which Alice sends funds. Then Alices’ transactions are executed and Alice transfers funds to Bob’s controlled VaultBooster.

This is a Medium severity issue that has been referenced from below Code4rena reports,
https://code4rena.com/reports/2023-01-rabbithole/#m-01-questfactory-is-suspicious-of-the-reorg-attack
https://code4rena.com/reports/2023-04-frankencoin#m-14-re-org-attack-in-factory

## Proof of Concept
https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBoosterFactory.sol#L29

## Tools Used
Manual Review

## Recommended Mitigation Steps
Deploy such contracts via `create2` with `salt` that includes `msg.sender`


## Assessed type

Other