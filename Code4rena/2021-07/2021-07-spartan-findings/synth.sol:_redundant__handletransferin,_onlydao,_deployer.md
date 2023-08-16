## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Synth.sol: Redundant _handleTransferIn, onlyDAO, DEPLOYER](https://github.com/code-423n4/2021-07-spartan-findings/issues/47) 

# Handle

hickuphh3


# Vulnerability details

### Impact

`_handleTransferIn()`, `DEPLOYER` and `onlyDAO()` are defined but unused. Hence, they can be removed from the contract.

