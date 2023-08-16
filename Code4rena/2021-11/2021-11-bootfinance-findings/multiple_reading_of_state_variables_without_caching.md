## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [multiple reading of state variables without caching](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/9) 

# Handle

pants


# Vulnerability details

In InvestorDistribution line 103 you read investors[_investor] multiple times (only at this function investors[_investor] is read 6 times).
You could cache the value and call the cached value instead.
Investors x = investors[_investor];
And then use x.amount, x.claimed, etc.

