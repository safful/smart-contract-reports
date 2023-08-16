## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Cache factory.ownerSplit()](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/89) 

# Handle

pauliax


# Vulnerability details

## Impact
function handleFees calls factory.ownerSplit() twice. To save some gas and reduce the number of external calls, you should save the value after the first call and re-use it later.

## Recommended Mitigation Steps
Cache factory.ownerSplit() in a local variable and re-use it.

