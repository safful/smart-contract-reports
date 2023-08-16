## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [tend() can be simplified](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/27) 

# Handle

pauliax


# Vulnerability details

## Impact
because function tend() always reverts, you can remove authorization checks and modifiers to save some gas.


