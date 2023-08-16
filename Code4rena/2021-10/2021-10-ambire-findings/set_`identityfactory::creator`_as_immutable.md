## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Set `IdentityFactory::creator` as immutable](https://github.com/code-423n4/2021-10-ambire-findings/issues/6) 

# Handle

pmerkleplant


# Vulnerability details

## Impact
Variable `creator` in the `IdentityFactory` is never reset after
initialization in the constructor. Declaring it as immutable saves gas.

