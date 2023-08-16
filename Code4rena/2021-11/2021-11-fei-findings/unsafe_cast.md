## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [unsafe cast](https://github.com/code-423n4/2021-11-fei-findings/issues/151) 

# Handle

danb


# Vulnerability details

oracle.pcvStats returns newProtocolEquity as  int256, it is then casted to uint256 in recalculate. If it is possible that newProtocolEquity will be negative, consider using SafeCast instead.

