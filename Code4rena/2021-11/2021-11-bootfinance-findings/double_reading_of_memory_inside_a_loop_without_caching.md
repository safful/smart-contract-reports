## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [double reading of memory inside a loop without caching](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/13) 

# Handle

pants


# Vulnerability details

When double reading a variable from memory the gas efficient way is to cache it and use the cached value. 

For example in line 537, 538 of SwapUtils.sol you have two accesses to xp[i]. You can save xp[i] as local variable and then use it instead at the second time.

