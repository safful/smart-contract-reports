## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Redundant hardhat console import ](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/173) 

# Handle

pants


# Vulnerability details

You import "import "./hardhat/console.sol";" and all uses are commented.
You should also comment the import.
SwapUtils line 9

