## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing verification on `tokenInit`'s lock](https://github.com/code-423n4/2021-07-sherlock-findings/issues/105) 

# Handle

cmichel


# Vulnerability details

The `Gov.tokenInit` skips the underlying token check if the `_token` is SHERX:

```solidity
if (address(_token) != address(this)) {
  require(_lock.underlying() == _token, 'UNDERLYING');
}
```

## Impact
This check should still be performed even for `_token == address(this) // SHERX`, otherwise, the lock can have a different underlying and potentially pay out wrong tokens.

## Recommendation
Verify the underlying of all locks.

