## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Short the following require messages](https://github.com/code-423n4/2021-12-pooltogether-findings/issues/13) 

# Handle

robee


# Vulnerability details

The following require messages are of length more than 32 and we think are short enough to short
them into exactly 32 characters such that it will be placed in one slot of memory and the require 
function will cost less gas. 
The list: 

        Solidity file: TwabRewards.sol, In line 128, Require message length to shorten: 38, The message: TwabRewards/recipient-not-zero-address
        Solidity file: TwabRewards.sol, In line 175, Require message length to shorten: 35, The message: TwabRewards/rewards-already-claimed
        Solidity file: TwabRewards.sol, In line 231, Require message length to shorten: 35, The message: TwabRewards/ticket-not-zero-address


