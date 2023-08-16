## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [onlyOracle never used](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/80) 

# Handle

pauliax


# Vulnerability details

## Impact
modifier onlyOracle in WrappedIbbtc is never used, so can be removed to reduce deployment gas costs.


