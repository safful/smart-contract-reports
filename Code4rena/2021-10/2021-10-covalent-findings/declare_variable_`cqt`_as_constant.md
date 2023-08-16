## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Declare variable `CQT` as constant](https://github.com/code-423n4/2021-10-covalent-findings/issues/67) 

# Handle

pmerkleplant


# Vulnerability details

## Impact
The variable `CQT` is used as constant but not declared as such.

Declaring it as constant saves gas.

