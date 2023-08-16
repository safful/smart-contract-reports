## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- float-wont-fix

# [LongShort.sol: Inconsistency in _claimAndDistributeYieldThenRebalanceMarket()](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/59) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The comparison and effecting `valueChange` in `_claimAndDistributeYieldThenRebalanceMarket()` can be made consistent with the other functions.

```jsx
if (valueChange > 0) {
	longValue += uint256(valueChange);
	shortValue -= uint256(valueChange);
} else {
  longValue -= uint256(-valueChange);
  shortValue += uint256(-valueChange);
}
```

### Recommended Mitigation Steps

Change the `else` case to `else if (valueChange < 0)`.

```jsx
if (valueChange > 0) {
	longValue += uint256(valueChange);
	shortValue -= uint256(valueChange);
} else if (valueChange < 0) {
  longValue -= uint256(-valueChange);
  shortValue += uint256(-valueChange);
}
```

