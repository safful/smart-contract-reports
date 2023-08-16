## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- ATokenYieldSource

# [Gas savings on uninitialized variables.](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/101) 

# Handle

tensors


# Vulnerability details

##Impact
Uninitialized variables initialize to 0 automatically. No need to explicitly initialize it. 

##Proof of concept
https://github.com/pooltogether/aave-yield-source/blob/bc65c875f62235b7af55ede92231a495ba091a47/contracts/yield-source/ATokenYieldSource.sol#L141

##Recommended mitigation steps
Replace with: `uint256 shares;`

