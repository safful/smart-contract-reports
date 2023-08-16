## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- sponsor disputed

# [Title: Double reading from calldata o](https://github.com/code-423n4/2021-09-swivel-findings/issues/43) 

# Handle

pants


# Vulnerability details

At both line 59 and 61 the code reads o[i]. It happens inside a loop therefore the best practice which is more gas efficient is to cache o[i] once and use it instead.

https://github.com/Swivel-Finance/gost/blob/v2/test/swivel/Swivel.sol#L59

## Tool Used
Manual code review.


