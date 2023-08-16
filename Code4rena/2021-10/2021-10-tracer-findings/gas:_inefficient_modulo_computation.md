## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Inefficient modulo computation](https://github.com/code-423n4/2021-10-tracer-findings/issues/22) 

# Handle

cmichel


# Vulnerability details

`PoolFactory.uint2str` computes `i % 10` as `uint8(_i - (_i / 10) * 10)`.
This intuitively seems more gas-expensive than doing `i % 10`.
Consider using `i % 10` instead which also makes the code simpler to read.


