## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Check if amount is not zero](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/82) 

# Handle

pauliax


# Vulnerability details

## Impact
functions mint, burn, transfer and transferFrom could skip other steps if the amount is 0.


