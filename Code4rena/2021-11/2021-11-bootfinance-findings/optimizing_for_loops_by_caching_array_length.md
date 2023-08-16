## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [optimizing for loops by caching array length](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/14) 

# Handle

pants


# Vulnerability details

optimizing for loops by caching memory array length, instead of calling it every time.

For example at Swap.sol at time 158 you should have

uint8 len = _pooledTokens.length
and in the next line define the forloop with stop condition of  i<len

This appears in many places in the code. At some places you did cached the array length correctly.


