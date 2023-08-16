## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [`PrizeSplit.sol#_totalPrizeSplitPercentageAmount()` Avoid unnecessary copy from storage to memory can save gas](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/40) 

# Handle

WatchPug


# Vulnerability details

https://github.com/pooltogether/v4-core/blob/055335bf9b09e3f4bbe11a788710dd04d827bf37/contracts/prize-strategy/PrizeSplit.sol#L135-L136

```solidity
PrizeSplitConfig memory split = _prizeSplits[index];
_tempTotalPercentage = _tempTotalPercentage + split.percentage;
```

Only `percentage` of the `PrizeSplitConfig` struct is accessed, however, the current implementation created a memory variable that will load `_prizeSplits[index]` and copy to memory, this is unnecessary and gas inefficient.

### Recommendation

Change to:

```solidity
_tempTotalPercentage = _tempTotalPercentage + split.percentage;
```

