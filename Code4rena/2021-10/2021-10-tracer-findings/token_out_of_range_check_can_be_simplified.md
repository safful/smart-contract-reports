## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [token out of range check can be simplified](https://github.com/code-423n4/2021-10-tracer-findings/issues/37) 

# Handle

pauliax


# Vulnerability details

## Impact
Because 'token' is of type uint here, this comparison can be simplified to reduce gas costs:
  require(token == 0 || token == 1, "Pool: token out of range"); //before

## Recommended Mitigation Steps
  require(token < 2, "Pool: token out of range"); //after

