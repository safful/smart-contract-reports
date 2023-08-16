## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [internal functions could be set private](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/18) 

# Handle

pants


# Vulnerability details

Since no contract inherent from SwapUtils all internal functions could be set private. For example  getD could be set private to save gas.

