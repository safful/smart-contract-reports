## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Unnecessary msg.sender != 0 check](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/218) 

# Handle

cmichel


# Vulnerability details

The `AirdropDistribution.claimExact` and `InvestorDistribution.claimExact` functions check that `msg.sender != address(0)`.

This is always true, nobody has the private key of the zero address and it cannot be spoofed.
This check can be removed.

