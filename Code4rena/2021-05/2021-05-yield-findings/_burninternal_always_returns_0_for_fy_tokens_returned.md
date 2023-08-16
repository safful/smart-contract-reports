## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [_burnInternal always returns 0 for fy tokens returned](https://github.com/code-423n4/2021-05-yield-findings/issues/35) 

# Handle

pauliax


# Vulnerability details

## Impact
Function _burnInternal always returns 0 as a third parameter. It should return tokensBurnt, tokenOut, fyTokenOut.

## Recommended Mitigation Steps
return (tokensBurned, tokenOut, fyTokenOut);

