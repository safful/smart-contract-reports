## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed

# [Vault will stop participating in draws in case if they deposited maximum assets to the underlying vault](https://github.com/code-423n4/2023-08-pooltogether-mitigation-findings/issues/67) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault/blob/main/src/Vault.sol#L1399


# Vulnerability details

## Impact
Vault will stop participating in draws in case if they deposited maximum assets to the underlying vault.

## Proof of Concept
Vault contract [has `maxMint` function](https://github.com/GenerationSoftware/pt-v5-vault/blob/main/src/Vault.sol#L506-L515). This function first checks allowed amount to mint in the PtVault and then also [checks amount allowed in underlying vault](https://github.com/GenerationSoftware/pt-v5-vault/blob/main/src/Vault.sol#L512-L514). It is very likely that this value will be less than in PtVault as some vaults can have restriction for 1 staker.

This means that once such amount of shares is minted, then [it's not allowed to mint anymore](https://github.com/GenerationSoftware/pt-v5-vault/blob/main/src/Vault.sol#L1398-L1399), so each deposit/mint will revert.

In order to contribute earned yield to the prize pool, liquidation pair should call `liquidate` function.
It's the only way to send earned yield to the prize pool. The process is next: liquidator will provide POOl tokens on behalf of Vault and Vault [should mint corresponding amount of shares](https://github.com/GenerationSoftware/pt-v5-vault/blob/main/src/Vault.sol#L783) for liquidator. In case if this call will be done after max amount is minted, then it will revert, which means that it will be not possible to send earned yield to the pool and earn prizes. In this moment the main purpose of vault stopped working: users earned yield to participate in prize distributions, but they can't do that.

Of course, when someone will withdraw, then it will be possible to liquidate again, but that will lead to bad user experience and also someone can ddos pool, by depositing again to not allow send contributions to the prize pool.

Another thing that is related to this issue is that in case if maxMint was reached, then `mintYieldFee` can't be called as well, as it also mints shares.
## Tools Used
VsCode
## Recommended Mitigation Steps
Maybe it will be best to not check for maxMint for liquidations and fee minting.


## Assessed type

Error