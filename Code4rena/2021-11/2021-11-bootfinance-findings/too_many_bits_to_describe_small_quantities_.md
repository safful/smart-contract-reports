## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [too many bits to describe small quantities ](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/8) 

# Handle

pants


# Vulnerability details

In InvestorDistribution you generally use uint256 for every quantity, although 256 bits are much more than necessary. For example for Investor structure you can change all to uint128, and for every normal use it will not affect and save gas.

