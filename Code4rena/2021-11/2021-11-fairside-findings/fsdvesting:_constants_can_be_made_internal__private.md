## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [FSDVesting: Constants can be made internal / private](https://github.com/code-423n4/2021-11-fairside-findings/issues/29) 

# Handle

hickuphh3


# Vulnerability details

## Impact

Since the defined constants are unneeded elsewhere, it can be defined to be `internal` or `private` to save gas.

## Recommended Mitigation Steps

```jsx
// One month in seconds
uint256 internal constant ONE_MONTH = 30 days;
// Cliff period for a vest
uint256 internal constant CLIFF = 12 * ONE_MONTH;
// Duration of a vest
uint256 internal constant DURATION = 30 * ONE_MONTH;
```

