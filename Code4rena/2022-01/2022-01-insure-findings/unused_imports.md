## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Unused imports](https://github.com/code-423n4/2022-01-insure-findings/issues/1) 

# Handle

robee


# Vulnerability details

In the following files there are contract imports that aren't used. 
Import of unnecessary files costs deployment gas (and is a bad coding practice that is important to ignore). 
The following is a full list of all unused imports, we went through the whole code to find it :) <solidity file> <line number> <actual import line>: 

        Factory.sol, line 13, import "hardhat/console.sol";
        IndexTemplate.sol, line 6, import "hardhat/console.sol";
        Parameters.sol, line 12, import "hardhat/console.sol";
        BondingPremium.sol, line 9, import "@openzeppelin/contracts/utils/math/SafeMath.sol";


