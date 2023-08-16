## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- fixed-in-upstream-repo

# [Docstring](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/27) 

# Handle

evertkors


# Vulnerability details

A lot of docstrings for marketIndex are ` @param marketIndex An int32 which uniquely identifies a market.` but it is a `uint32` not an `int32`

