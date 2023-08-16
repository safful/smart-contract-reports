## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [uint is always >= 0](https://github.com/code-423n4/2021-08-notional-findings/issues/53) 

# Handle

pauliax


# Vulnerability details

## Impact
There are several checks that uint is not negative, e.g.:
  cashGroup.maxMarketIndex >= 0 &&
These checks are pretty much useless as uint can never be negative. Remove them to save some gas.


