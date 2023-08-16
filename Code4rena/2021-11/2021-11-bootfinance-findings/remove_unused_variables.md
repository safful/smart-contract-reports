## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Remove unused variables](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/161) 

# Handle

pmerkleplant


# Vulnerability details

Removing unused variables saves gas and increases code clarity.

Following variables are unused and can be removed:

- `Hour` in `vesting/contracts/AidropDistribution.sol`
- `Day` in `vesting/contracts/AirdropDistribution.sol`
- `Hour` in `vesting/contracts/InvestorDistribution.sol`
- `Day` in `vesting/contracts/InvestorDistribution.sol`
- `_balance` in `tge/contracts/PublicSale.sol`
- `_allowance` in `tge/contracts/PublicSale.sol`

