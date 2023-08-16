## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Repeated math operations](https://github.com/code-423n4/2022-01-insure-findings/issues/308) 

# Handle

pauliax


# Vulnerability details

## Impact
Can be refactored, from this:
```solidity
  require(
      request.timestamp +
          parameters.getLockup(msg.sender) <
          block.timestamp,
      "ERROR: WITHDRAWAL_QUEUE"
  );
  require(
      request.timestamp +
          parameters.getLockup(msg.sender) +
          parameters.getWithdrawable(msg.sender) >
          block.timestamp,
      "ERROR: WITHDRAWAL_NO_ACTIVE_REQUEST"
  );
```
to this:
```solidity
  uint256 unlocksAt = request.timestamp + parameters.getLockup(msg.sender);
  require(
      unlocksAt < block.timestamp,
      "ERROR: WITHDRAWAL_QUEUE"
  );
  require(
      unlocksAt + parameters.getWithdrawable(msg.sender) > block.timestamp,
      "ERROR: WITHDRAWAL_NO_ACTIVE_REQUEST"
  );
```

There are more places where this optimization could be applied besides the provided example, but the basic idea is to cache the result of repeated math operation when the value does not change.

