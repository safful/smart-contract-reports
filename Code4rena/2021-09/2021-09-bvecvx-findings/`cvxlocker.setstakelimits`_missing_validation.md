## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`CvxLocker.setStakeLimits` missing validation](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/50) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `CvxLocker.setStakeLimits` function does not check `_minimum <= _maximum`.

## Recommended Mitigation Steps
Implement these two checks instead:

```solidity
require(_minimum <= _maximum, "min range");
require(_maximum <= denominator, "max range");
```

