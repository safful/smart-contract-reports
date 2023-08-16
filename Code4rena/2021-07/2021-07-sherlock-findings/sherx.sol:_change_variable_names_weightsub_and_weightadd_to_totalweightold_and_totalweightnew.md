## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [SherX.sol: Change variable names weightSub and weightAdd to totalWeightOld and totalWeightNew](https://github.com/code-423n4/2021-07-sherlock-findings/issues/45) 

# Handle

hickuphh3


# Vulnerability details

### Impact

In `setWeights()`, the variables `weightAdd` and `weightSub` are used to ensure that there is no difference in the total weight.

### Recommended Mitigation Steps

Consider `totalWeightOld` and `totalWeightNew` as the variable names instead as they are more indicative of the intended usage and behaviour.

