## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Make variable veCVXStrategy::MAX_BPS constant](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/13) 

# Handle

pmerkleplant


# Vulnerability details

## Impact
Variable `MAX_BPS` in the veCVXStrategy is never reset after initialization. Declaring it as a constant saves gas.

## Tools Used
slither

