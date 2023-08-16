## Tags

- bug
- sponsor confirmed
- 1 (Low Risk)
- resolved

# [Hardcoded WETH](https://github.com/code-423n4/2021-10-ambire-findings/issues/54) 

# Handle

pauliax


# Vulnerability details

## Impact
WETH address is hardcoded but it may differ on other chains, e.g. Polygon, so make sure to check this before deploying and update if neccessary:
  address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

## Recommended Mitigation Steps
You should consider injecting WETH address via the constructor.

