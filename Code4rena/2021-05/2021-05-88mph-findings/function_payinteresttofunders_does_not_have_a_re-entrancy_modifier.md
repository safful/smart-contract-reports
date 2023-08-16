## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- resolved

# [function payInterestToFunders does not have a re-entrancy modifier](https://github.com/code-423n4/2021-05-88mph-findings/issues/6) 

# Handle

paulius.eth


# Vulnerability details

## Impact
function payInterestToFunders does not have a re-entrancy modifier. I expect to see this modifier because similar functions (including sponsored version) have it.

## Recommended Mitigation Steps
Add 'nonReentrant' to function payInterestToFunders.

