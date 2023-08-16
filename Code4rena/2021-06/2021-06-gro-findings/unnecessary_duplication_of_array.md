## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary duplication of array](https://github.com/code-423n4/2021-06-gro-findings/issues/27) 

# Handle

a_delamo


# Vulnerability details

## Impact

The methods `_stableToUsd` and `_stableToLp`  in the`Buoy3Pool.sol` contract is duplicating the array unnecessarily and costing gas to the users.

```
function _stableToUsd(uint256[N_COINS] memory tokenAmounts, bool deposit)
    internal
    view
    returns (uint256)
  {
    require(tokenAmounts.length == N_COINS, "deposit: !length");
    uint256[N_COINS] memory _tokenAmounts;
    for (uint256 i = 0; i < N_COINS; i++) {
      _tokenAmounts[i] = tokenAmounts[i];
    }
    uint256 lpAmount = curvePool.calc_token_amount(_tokenAmounts, deposit);
    return _lpToUsd(lpAmount);
  }

  function _stableToLp(uint256[N_COINS] memory tokenAmounts, bool deposit)
    internal
    view
    returns (uint256)
  {
    require(tokenAmounts.length == N_COINS, "deposit: !length");
    uint256[N_COINS] memory _tokenAmounts;
    for (uint256 i = 0; i < N_COINS; i++) {
      _tokenAmounts[i] = tokenAmounts[i];
    }
    return curvePool.calc_token_amount(_tokenAmounts, deposit);
  }
```

