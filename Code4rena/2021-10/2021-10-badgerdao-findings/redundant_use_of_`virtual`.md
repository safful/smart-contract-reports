## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Redundant use of `virtual`](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/64) 

# Handle

WatchPug


# Vulnerability details

Based on the context, the functions listed below are not expected to be overridden, thus the use of the keyword `virtual` is redundant.

- transfer()
- updatePricePerShare()
- transferFrom()
- pricePerShare()

### Recommendation

Consider removing `virtual` for these functions.

