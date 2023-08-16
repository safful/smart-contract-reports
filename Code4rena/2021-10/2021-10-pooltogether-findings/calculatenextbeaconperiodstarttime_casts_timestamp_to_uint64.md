## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- resolved

# [calculateNextBeaconPeriodStartTime casts timestamp to uint64](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/44) 

# Handle

pauliax


# Vulnerability details

## Impact
function calculateNextBeaconPeriodStartTime accepts _time as a type of uint256 but later explicitly casts it to uint64. While this function is not used internally, it behaves incorrectly when passed a value that uint64 does not hold (for such values it will return a max value of uint64). I don't see a reason why you can't directly accept uint64 here.

## Recommended Mitigation Steps
Change parameter type to uint64.

