## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [executionPrice, newMakeAverage and newTakeAverage before calling the market](https://github.com/code-423n4/2021-06-tracer-findings/issues/52) 

# Handle

pauliax


# Vulnerability details

## Impact
Trader function executeTrade calculates executionPrice, newMakeAverage, newTakeAverage, then calls the market, and only if it succeeds it uses these variables.

## Recommended Mitigation Steps
Better first call the market and only then calculate and use these variables to avoid useless calculations and gas costs.

