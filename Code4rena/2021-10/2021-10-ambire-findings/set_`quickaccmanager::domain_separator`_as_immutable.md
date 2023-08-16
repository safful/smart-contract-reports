## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Set `QuickAccManager::DOMAIN_SEPARATOR` as immutable](https://github.com/code-423n4/2021-10-ambire-findings/issues/4) 

# Handle

pmerkleplant


# Vulnerability details

## Impact
Variable `DOMAIN_SEPARATOR` in the `QuickAccManager` is never reset after
initialization in the constructor. Declaring it as immutable saves gas.

