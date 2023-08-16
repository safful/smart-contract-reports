## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# ["> 0" is less efficient than "!= 0" for unsigned integers](https://github.com/code-423n4/2021-12-maple-findings/issues/24) 

# Handle

ye0lde


# Vulnerability details

## Impact

Gas savings 

## Proof of Concept

"> 0" is used in the following location(s):

https://github.com/maple-labs/debt-locker/blob/81f55907db7b23d27e839b9f9f73282184ed4744/contracts/DebtLocker.sol#L312

## Tools Used
Visual Studio Code, Remix

## Recommended Mitigation Steps

Change "> 0" to "!=0" for small gas savings.



