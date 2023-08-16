## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Unused imports](https://github.com/code-423n4/2022-01-yield-findings/issues/8) 

# Handle

robee


# Vulnerability details

In the following files there are contract imports that aren't used. 
Import of unnecessary files costs deployment gas (and is a bad coding practice that is important to ignore). 

        ConvexModule.sol, line 3, import "@yield-protocol/vault-interfaces/DataTypes.sol";
        ConvexStakingWrapper.sol, line 9, import "./interfaces/IConvexDeposits.sol";
        ConvexStakingWrapper.sol, line 10, import "./interfaces/ICvx.sol";

