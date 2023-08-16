## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Useless nonReentrant](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/293) 

# Handle

pauliax


# Vulnerability details

## Impact
functions validate and modifyInvestor do not need nonReentrant modifier as they do not execute any external calls where you can hook up to re-enter.


