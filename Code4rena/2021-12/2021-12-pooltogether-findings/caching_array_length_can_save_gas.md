## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Caching array length can save gas](https://github.com/code-423n4/2021-12-pooltogether-findings/issues/16) 

# Handle

robee


# Vulnerability details

Caching the array length is more gas efficient. 
This is because access to a local variable in solidity is more efficient than query storage / calldata / memory 
We recommend to change from: 
for (uint256 i=0; i<array.length; i++) { ... }
to:
uint len = array.length 
 for (uint256 i=0; i<len; i++) { ... }
These functions use not using prefix increments (`++x`) or not using the unchecked keyword: 

        TwabRewards.sol, _epochIds, 172
        TwabRewards.sol, _epochIds, 217


