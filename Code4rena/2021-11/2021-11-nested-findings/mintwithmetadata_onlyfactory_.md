## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [mintWithMetadata onlyFactory ](https://github.com/code-423n4/2021-11-nested-findings/issues/213) 

# Handle

pauliax


# Vulnerability details

## Impact
function "mintWithMetadata" does not need onlyFactory modifier as it will be checked in function "mint" later.


