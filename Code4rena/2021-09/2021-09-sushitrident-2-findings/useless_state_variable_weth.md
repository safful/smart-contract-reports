## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Useless state variable wETH](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/73) 

# Handle

pauliax


# Vulnerability details

## Impact
contract ConcentratedLiquidityPosition has a state variable 'wETH' but it is not being used in any meaningful way. So you can remove it to save some gas.

## Recommended Mitigation Steps
Remove useless state variables.

