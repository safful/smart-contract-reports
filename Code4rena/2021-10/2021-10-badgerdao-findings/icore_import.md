## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [ICore import](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/79) 

# Handle

pauliax


# Vulnerability details

## Impact
You import ICore interface but actually need only one function from it: pricePerShare(). Consider importing a minimal ICore interface with only the functions that you actually use to reduce deployment gas costs. Or you can just simply re-use ICoreOracle.


