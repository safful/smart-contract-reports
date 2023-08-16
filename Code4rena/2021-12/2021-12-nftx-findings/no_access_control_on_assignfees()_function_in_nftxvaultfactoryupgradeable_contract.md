## Tags

- bug
- 2 (Med Risk)
- resolved
- sponsor confirmed

# [No access control on assignFees() function in NFTXVaultFactoryUpgradeable contract](https://github.com/code-423n4/2021-12-nftx-findings/issues/50) 

# Handle

ych18


# Vulnerability details

In If the Vault owner decides to set factoryMintFee and factoryRandomRedeemFee to zero, any user could call the function NFTXVaultFactoryUpgradeable.assignFees() and hence all the fees are updated.

