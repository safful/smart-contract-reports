## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [_isBeforeFeeBootstrapPeriod inside the loop](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/296) 

# Handle

pauliax


# Vulnerability details

## Impact
_isBeforeFeeBootstrapPeriod() is re-evaluated again and again inside the loop, although its value could be cached outside the loop and re-used to reduce gas costs.


