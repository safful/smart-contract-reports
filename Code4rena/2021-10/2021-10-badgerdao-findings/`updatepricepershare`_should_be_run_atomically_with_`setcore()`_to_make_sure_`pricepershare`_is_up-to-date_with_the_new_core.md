## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`updatePricePerShare` should be run atomically with `setCore()` to make sure `pricePerShare` is up-to-date with the new Core](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/61) 

# Handle

WatchPug


# Vulnerability details

Given that `setCore()` could potentially lead to a change of `pricePerShare`, and `pricePerShare` will not be updated until `updatePricePerShare()` is called separately.

To ensure `pricePerShare` is up-to-date, `updatePricePerShare` should be run atomically with `setCore()`.

### Recommendation

Consider changing `setCore()` to:

```solidity
function setCore(address _core) external onlyGovernance {
    core = ICore(_core);

    updatePricePerShare();

    emit SetCore(_core);
}
```

