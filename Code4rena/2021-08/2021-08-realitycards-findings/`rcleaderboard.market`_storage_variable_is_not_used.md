## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`RCLeaderboard.market` storage variable is not used](https://github.com/code-423n4/2021-08-realitycards-findings/issues/55) 

# Handle

cmichel


# Vulnerability details

The `RCLeaderboard.market` storage variable is never used.
Instead, the `MARKET` role seems to be used to implement authentication.

## Impact
Unused code can hint at programming or architectural errors.

## Recommended Mitigation Steps
Use it or remove it.

