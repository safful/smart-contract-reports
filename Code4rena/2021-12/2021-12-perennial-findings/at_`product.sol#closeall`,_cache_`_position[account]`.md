## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [At `Product.sol#closeAll`, cache `_position[account]`](https://github.com/code-423n4/2021-12-perennial-findings/issues/65) 

# Handle

0x0x0x


# Vulnerability details

At `Product.sol#closeAll` cache `_position[account]` to save gas. 

In the first line of the function `_position[account]` is used twice and gas can be saved by caching it.

