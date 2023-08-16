## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Contract `Vesting` should inherit from interface `IVesting`](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/164) 

# Handle

pmerkleplant


# Vulnerability details

Contract `Vesting` in `vesting/contracts/Vesting.sol` should inherit from the
interface `IVesting` in `vesting/contracts/interfaces/IVesting.sol` as the
contract implements the interface.

