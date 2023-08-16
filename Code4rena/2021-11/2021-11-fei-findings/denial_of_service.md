## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [denial of service](https://github.com/code-423n4/2021-11-fei-findings/issues/150) 

# Handle

danb


# Vulnerability details

## Impact
in the first call to requery, If the oracle returns newProtocolEquity = 0, it can never be changed and would lead to denial of service of the system.
## Proof of Concept

In requery, init is checked to be false if newProtocolEquity = 0, and then set to true. so if it is already initialized and newProtocolEquity = 0, it wouldn't change anything
## Tools Used
manual review

