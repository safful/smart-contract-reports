## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- disagree with severity

# ['peek' and 'get' are identical (non-transactional)](https://github.com/code-423n4/2021-05-yield-findings/issues/19) 

# Handle

pauliax


# Vulnerability details

## Impact
In the contract ChainlinkMultiOracle both functions 'peek' and 'get' are identical. They are declared as views while based on IOracle interface 'get' should be transactional.


