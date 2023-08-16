## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [function _getPrizeSplitAmount can be refactored](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/48) 

# Handle

pauliax


# Vulnerability details

## Impact
If you want to save some gas you can get rid of _getPrizeSplitAmount and calculate the split directly in _distributePrizeSplits as this function is internal and is only called once so there is no actual need for reusability here and removing this extra call will make the execution cheaper.

## Recommended Mitigation Steps
Consider moving the logic of _getPrizeSplitAmount  directly to _distributePrizeSplits.

