## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Pack structs tightly](https://github.com/code-423n4/2021-10-mochi-findings/issues/153) 

# Handle

pauliax


# Vulnerability details

## Impact
Gas efficiency can be achieved by tightly packing the struct. Struct variables are stored in 32 bytes each so you can group smaller types to occupy less storage. For example, startedAt or boughtAt in Auction struct hold block.number so realistically this does not need uint256 and you can consider storing it in lower type. You can read more here: https://fravoll.github.io/solidity-patterns/tight_variable_packing.html or in the official documentation: https://docs.soliditylang.org/en/v0.4.21/miscellaneous.html

## Recommended Mitigation Steps
Search for an optimal size and order of structs to reduce gas usage.

