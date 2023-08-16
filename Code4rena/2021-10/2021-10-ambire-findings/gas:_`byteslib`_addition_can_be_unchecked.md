## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Gas: `BytesLib` addition can be unchecked](https://github.com/code-423n4/2021-10-ambire-findings/issues/41) 

# Handle

cmichel


# Vulnerability details

The `index += 32` addition in `readBytes32` can be put in an `unsafe` block as the array length is already checked to be greater than the addition.

