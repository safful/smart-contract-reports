## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Inconsistency in pragma solidity version definition in InsureDAOERC20.sol](https://github.com/code-423n4/2022-01-insure-findings/issues/242) 

# Handle

hubble


# Vulnerability details


## Impact
Inconsistency in pragma solidity versions in different solidity files.

## Proof of Concept
File : InsureDAOERC20.sol
       pragma solidity ^0.8.0;

All other solidity files in the project
       pragma solidity 0.8.7;

## Tools Used
Manual review

## Recommended Mitigation Steps
Set the version to 0.8.7 in the InsureDAOERC20.sol file

