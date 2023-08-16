## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Inaccurate revert messages](https://github.com/code-423n4/2021-11-malt-findings/issues/343) 

# Handle

pauliax


# Vulnerability details

## Impact
Inaccurate revert messages:
```solidity        
  _delay >= 0 && _delay < gracePeriod,
  "Timelock::setDelay: Delay must not be greater equal to zero and less than gracePeriod"

  require(startEpoch < endEpoch, "Start cannot be before the end");
  require(rewardAmount <= rewardEarned, "< earned");
  require(bondedBalance > 0, "< bonded balance");
  require(amount <= bondedBalance, "< bonded balance");
```


