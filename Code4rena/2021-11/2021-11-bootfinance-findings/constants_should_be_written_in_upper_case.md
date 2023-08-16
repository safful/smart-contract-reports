## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Constants should be written in UPPER_CASE](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/165) 

# Handle

pmerkleplant


# Vulnerability details

Constants should be written in UPPER_CASE, see [Solidity naming conventions](https://docs.soliditylang.org/en/v0.4.25/style-guide.html#constants).

Constants breaking this convention:

- `decimals` in `tge/contracts/PublicSale.sol`
- `coin` in `tge/contracts/PublicSale.sol`
- `secondsPerDay` in `tge/contracts/PublicSale.sol`
- `firstEra` in `tge/contracts/PublicSale.sol`

