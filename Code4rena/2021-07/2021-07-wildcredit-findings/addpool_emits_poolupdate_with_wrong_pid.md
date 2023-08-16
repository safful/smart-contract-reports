## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [addPool emits PoolUpdate with wrong pid](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/68) 

# Handle

pauliax


# Vulnerability details

## Impact
Function addPool emits event PoolUpdate passing pools.length as pid while the actual pid is pools.length-1.

## Recommended Mitigation Steps
   emit PoolUpdate(pools.length-1, _pair, _token, _isSupply, _points);
or even better store it in a temporary variable and re-use multiple times.

