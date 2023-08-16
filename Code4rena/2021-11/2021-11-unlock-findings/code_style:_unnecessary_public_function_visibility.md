## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Code Style: Unnecessary public function visibility](https://github.com/code-423n4/2021-11-unlock-findings/issues/184) 

# Handle

WatchPug


# Vulnerability details

It's a best practice to limit the visibility to `external` if the function is expected to be called externally only.

```solidity=180{185}
  /**
   * Public function which returns the total number of unique owners (both expired
   * and valid).  This may be larger than totalSupply.
   */
  function numberOfOwners()
    public
    view
    returns (uint)
  {
    return owners.length;
  }
```

`numberOfOwners()` can be changed to `external`.

