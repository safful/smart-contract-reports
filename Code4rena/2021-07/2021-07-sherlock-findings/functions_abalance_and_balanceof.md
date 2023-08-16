## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Functions aBalance and balanceOf](https://github.com/code-423n4/2021-07-sherlock-findings/issues/72) 

# Handle

pauliax


# Vulnerability details

## Impact
There is no difference between functions aBalance and balanceOf in contract AaveV2, they both return aWant, so there is no point in having them separately.

## Recommended Mitigation Steps
Remove internal function aBalance and make balanceOf public.

