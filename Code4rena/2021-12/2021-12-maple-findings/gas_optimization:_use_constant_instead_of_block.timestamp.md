## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas Optimization: Use constant instead of block.timestamp](https://github.com/code-423n4/2021-12-maple-findings/issues/55) 

# Handle

gzeon


# Vulnerability details

## Impact
Use type(uint).max instead of block.timestamp to save gas

https://github.com/maple-labs/liquidations/blob/bb09e17b1fac1126ce7734e58c3133be06162590/contracts/UniswapV2Strategy.sol#L71
https://github.com/maple-labs/liquidations/blob/bb09e17b1fac1126ce7734e58c3133be06162590/contracts/SushiswapStrategy.sol#L71

