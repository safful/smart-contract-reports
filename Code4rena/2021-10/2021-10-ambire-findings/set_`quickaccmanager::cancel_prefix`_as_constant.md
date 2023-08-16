## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Set `QuickAccManager::CANCEL_PREFIX` as constant](https://github.com/code-423n4/2021-10-ambire-findings/issues/7) 

# Handle

pmerkleplant


# Vulnerability details

## Impact
Variable `CANCEL_PREFIX` in the `QuickAccManager` is never reset after
initialization. Declaring it as a constant saves gas.

