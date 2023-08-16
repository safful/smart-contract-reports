## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Loss of precision and increased gas cost with double assignment on a calculation  ](https://github.com/code-423n4/2022-01-insure-findings/issues/84) 

# Handle

Dravee


# Vulnerability details

In `IndexTemplate.sol:withdrawable()`, the following can be optimized to save gas and avoid a loss of precision, from:
```
                uint256 _necessaryAmount = _targetLockedCreditScore * totalAllocPoint /  _targetAllocPoint;
                _necessaryAmount = _necessaryAmount *  MAGIC_SCALE_1E6 / targetLev;
```
to
```
                uint256 _necessaryAmount = _targetLockedCreditScore * totalAllocPoint *  MAGIC_SCALE_1E6 /  (_targetAllocPoint * targetLev);
```

