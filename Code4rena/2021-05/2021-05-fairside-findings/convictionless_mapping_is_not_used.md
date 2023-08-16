## Tags

- bug
- question
- 1 (Low Risk)
- sponsor confirmed
- resolved

# [convictionless mapping is not used](https://github.com/code-423n4/2021-05-fairside-findings/issues/61) 

# Handle

pauliax


# Vulnerability details

## Impact
convictionless can be set via function setConvictionless, however, it is not used anywhere across the system, thus making it useless. Based on the comment above this variable, I expect to see it used in functions like _updateConvictionScore.

## Recommended Mitigation Steps
Either remove this mapping or use it where intended.

