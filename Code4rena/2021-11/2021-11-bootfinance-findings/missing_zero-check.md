## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing Zero-check](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/146) 

# Handle

0v3rf10w


# Vulnerability details

## Impact
Missing Zero address check

## Proof of Concept
BasicSale.constructor(IERC20,IERC721,IVesting,uint256,uint256,uint256,uint256,address)._burnAddress (tge/contracts/PublicSale.sol#112) 
lacks a zero-check on :-
burnAddress = _burnAddress (tge/contracts/PublicSale.sol#137)

## Tools Used
Manual

## Recommended Mitigation Steps
Check that the address is zero 

