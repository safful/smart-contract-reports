## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Events should be written in CapWords](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/162) 

# Handle

pmerkleplant


# Vulnerability details

Events should be written in CapWords, see [Solidity naming conventions](https://docs.soliditylang.org/en/v0.4.25/style-guide.html#event-names).

Events breaking this convention:

- `updateMiningParameters` in `vesting/contracts/AidropDistribution.sol`
- `updateMiningParameters` in `vesting/contracts/InvestorDistribution.sol`

