## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`BasicSale` uses inaccurate `secondsPerDay` value](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/211) 

# Handle

cmichel


# Vulnerability details

The `BasicSale` contract uses a `secondsPerDay` value of `84200` but one day has `86400` seconds.

## Impact
The `secondsPerDay` does not reflect seconds per day.

## Recommended Mitigation Steps
Change the value.

