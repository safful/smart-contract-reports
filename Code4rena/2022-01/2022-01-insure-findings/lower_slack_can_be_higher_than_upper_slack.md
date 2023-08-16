## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed
- sponsor disputed

# [Lower slack can be higher than upper slack](https://github.com/code-423n4/2022-01-insure-findings/issues/248) 

# Handle

cmichel


# Vulnerability details

The `Parameters.setLowerSlack/setUpperSlack` functions do not check that the new value does still satisfy the `_lowerSlack <= _upperSlack` condition.

## Recommended Mitigation Steps
Check that `_lowerSlack <= _upperSlack`  is still satisfied in these functions.


