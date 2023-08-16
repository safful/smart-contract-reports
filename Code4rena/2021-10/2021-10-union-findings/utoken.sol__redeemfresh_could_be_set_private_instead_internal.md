## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [UToken.sol _redeemFresh could be set private instead internal](https://github.com/code-423n4/2021-10-union-findings/issues/95) 

# Handle

pants


# Vulnerability details

This is one of many examples of the appearance of private instead of internal. Since we manually code reviewing and writing issues we don't list all the appearances.
Calling a private function is more gas efficient than calling internal. 

Here we refer to UToken.sol._redeemFresh function that is used only in UToken.sol file. 

