## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [_processPerformanceFees is useless now](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/24) 

# Handle

pauliax


# Vulnerability details

## Impact
function _processPerformanceFees is not used. functions _processPerformanceFees and _processRewardsFees are way too similar. _processPerformanceFees can be eliminated and _processRewardsFees used by passing want as a _token parameter.

