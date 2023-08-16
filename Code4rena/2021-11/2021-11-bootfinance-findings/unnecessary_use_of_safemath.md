## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary use of safeMath](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/7) 

# Handle

pants


# Vulnerability details

In InvestorDistribution at line 19 you have

    using SafeMath for uint256;

although you use solidity >0.8.0
therefore you don't need to use safeMath for uint256

