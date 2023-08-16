## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Repetitive storage access](https://github.com/code-423n4/2021-05-fairside-findings/issues/15) 

# Handle

pauliax


# Vulnerability details

## Impact
function _addTribute can reuse lastTribute to reduce the numbers of storage access: tributes[totalTributes - 1].amount = add224(...) can be replaced with lastTribute.amount = add224(...) as it is already a storage pointer that can be assigned a value with no need to recalculate the index and access the array again. Same situation with function _addGovernanceTribute governanceTributes.

## Recommended Mitigation Steps
lastTribute.amount = add224(...)

