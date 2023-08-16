## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unused import](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/72) 

# Handle

pauliax


# Vulnerability details

## Impact
There is an unused import: import "../../interfaces/ITridentRouter.sol"; in ConcentratedLiquidityPosition. It will increase the size of deployment with no real benefit.

## Recommended Mitigation Steps
Consider removing this unused import to save some gas.


