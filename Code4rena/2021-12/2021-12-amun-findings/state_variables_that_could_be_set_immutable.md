## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [State variables that could be set immutable](https://github.com/code-423n4/2021-12-amun-findings/issues/18) 

# Handle

robee


# Vulnerability details

In the following files there are state variables that could be set immutable to save gas. 
The list of format <solidity file>, <state variable name that could be immutable>: 

        basket in RebalanceManagerV2.sol
        uniSwapLikeRouter in SingleNativeTokenExit.sol
        INTERMEDIATE_TOKEN in SingleNativeTokenExit.sol
        uniSwapLikeRouter in SingleNativeTokenExitV2.sol
        INTERMEDIATE_TOKEN in SingleNativeTokenExitV2.sol
        uniSwapLikeRouter in SingleTokenJoin.sol
        INTERMEDIATE_TOKEN in SingleTokenJoin.sol
        uniSwapLikeRouter in SingleTokenJoinV2.sol
        INTERMEDIATE_TOKEN in SingleTokenJoinV2.sol
        predicateProxy in MintableERC20.sol
        underlying in PolygonERC20Wrapper.sol
        childChainManager in PolygonERC20Wrapper.sol

