## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Dao.sol: Reserve emissions must be turned on for depositLPs and bonds](https://github.com/code-423n4/2021-07-spartan-findings/issues/44) 

# Handle

hickuphh3


# Vulnerability details

### Impact

`depositLPForMember()` and `bond()` invokes `harvest()` if a user has existing LP deposits or bonded assets into the DAO. This is to prevent users from depositing more assets before calling `harvest()` to earn more DAOVault incentives. However, `harvest()` reverts if reserve emissions are turned off. 

Hence, deposits / bonds performed by existing users will fail should reserve emissions be disabled.

### Recommended Mitigation Steps

Cache claimable rewards into a separate mapping when `depositLPForMember()` and `bond()` are called. `harvest()` will then attempt to claim these cached + pending rewards. Perhaps Synthetix's Staking Rewards contract or Sushiswap's FairLaunch contract can provide some inspiration.

