## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Approve `cToken` address only once for underlying](https://github.com/code-423n4/2021-09-swivel-findings/issues/87) 

# Handle

cmichel


# Vulnerability details

The `Swivel` contract approves the `cToken` address whenever it wants to mint `cTokens` upon order fulfilment, see `initiateVaultFillingZcTokenInitiate`.

It could just approve the `cToken` contract once for each market underlying it supports with the max value and save the approval call for each order fulfilment.


