## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [_burnNST](https://github.com/code-423n4/2021-11-nested-findings/issues/208) 

# Handle

pauliax


# Vulnerability details

## Impact
I think it is not necessary to have function _burnNST as a separate private function. It is called only once and has just one LOC so it just incurs in extra gas cost which can be avoided by moving this line to function trigger and getting rid of _burnNST.


