## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [NestedAsset.setFactory should be named addFactory](https://github.com/code-423n4/2021-11-nested-findings/issues/204) 

# Handle

hyh


# Vulnerability details

```setFactory``` should be named ```addFactory``` as it doesn't set the only factory, but adds to the list of factories

https://github.com/code-423n4/2021-11-nested/blob/main/contracts/NestedAsset.sol#L133

