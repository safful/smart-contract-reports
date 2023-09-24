## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-37

# [Branch Strategies lose yield due to wrong implementation of time limit in BranchPort.sol](https://github.com/code-423n4/2023-05-maia-findings/issues/264) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchPort.sol#L193-L199


# Vulnerability details

## Impact
Branch Strategies lose yield due to a wrong implementation of the time limit in BranchPort.sol. This results in missed yield for branch strategies, less capital utilization of the platform, and ultimately a loss of additional revenue for the protocol's users.

## Proof of Concept

The `_checkTimeLimit` function in `BranchPort.sol` controls whether amounts used by a branch strategy do cumulatively not exceed the daily limit which is set for the particular strategy. It is only called from the `manage` function in the same contract (https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchPort.sol#L161).

The current implementation of `_checkTimeLimit` looks like this:

```Solidity
function _checkTimeLimit(address _token, uint256 _amount) internal {
    if (block.timestamp - lastManaged[msg.sender][_token] >= 1 days) {
        strategyDailyLimitRemaining[msg.sender][_token] = strategyDailyLimitAmount[msg.sender][_token];
    }
    strategyDailyLimitRemaining[msg.sender][_token] -= _amount;
    lastManaged[msg.sender][_token] = block.timestamp;
}
```

**I.) The current implementation does the following**

1. The first time a strategy manages some amount and `_checkTimeLimit` is called the 24h window is started (strategyDailyLimitRemaining[msg.sender][_token] is initialized to the daily limit amount and lastManaged[msg.sender][_token] is set to block.timestamp)

2. On a second call to use more of the daily limit (if the amount used in 1. is not the full daily amount, which is not enforced), it will set lastManaged[msg.sender][_token] again to block.timestamp. This pushes the time when the daily budget will be reset (strategyDailyLimitRemaining[msg.sender][_token] = strategyDailyLimitAmount[msg.sender][_token]) again 24 hours into the future.

**Consequences of the current implementation**

- Due to the setting of the lastManaged[msg.sender][_token] on every call the daily budget misses its purpose as a budget reset after 24h is not guaranteed.

- In the worst but likely case, a call is made by the strategy just before the current 24h time window passes to use the remaining amount. This will delay a reset of the daily limit by the maximum possible time. In consequence, a strategy misses 1 full amount of the daily budget.

- The aforementioned results in a loss of yield for the strategy (assuming the strategy generates a yield), less capital utilization of the platform, and ultimately a loss of additional revenue for the protocol's users.

- Assuming there are multiple strategies in the protocol, the negative effect is multiplied.

**II.) The implementation that was probably intended**

```Solidity
function _checkTimeLimit(address _token, uint256 _amount) internal {
    if (block.timestamp - lastManaged[msg.sender][_token] >= 1 days) {
        strategyDailyLimitRemaining[msg.sender][_token] = strategyDailyLimitAmount[msg.sender][_token];
        lastManaged[msg.sender][_token] = block.timestamp; // <--- line moved here
    }
    strategyDailyLimitRemaining[msg.sender][_token] -= _amount;
}
```

- Here the reset of the daily budget is made after a 24h time window as expected.

- What is lost, is the information "when the last time a strategy called the function" as lastManaged[msg.sender][_token] now only stores the block timestamp the last time the daily budget was reset and not when the last time the function was called. If this should still be tracked consider an additional state variable (e.g. lastDailyBudgetReset[msg.sender][_token]).

## Tools Used
Manual review

## Recommended Mitigation Steps

Implement the logic as shown under "II.) The implementation that was probably intended".

Please also, consider the following comments:

- To get the maximum amount out of their daily budget a strategy must make a call to the `manage()` function exactly every 24h hours after the first time calling it. Otherwise, there are time frames where amounts could be retrieved but are not. That would have the strategy missing out on investments and therefore potential yield. E.g. 2nd call happens 36h (instead of 24h) after the initial call => 12 hours (1/2 of a daily budget) remains unused.

- The amount also needs to be fully used within the 24h timeframe since the daily limit is overwriting and not cumulating (using strategyDailyLimitRemaining[msg.sender][_token] = strategyDailyLimitAmount[msg.sender][_token] and not strategyDailyLimitRemaining[msg.sender][_token] += strategyDailyLimitAmount[msg.sender][_token])

- An alternative to the aforementioned could be to calculate the amount to grant to a strategy after an initial/last grant like the following: (time since last grant of fresh daily limit / 24h) * daily limit. This would have the effect that a strategy could use their granted limits without missing amounts due to suboptimal timing. It would also spare the strategy the necessary call every 24h which would save some gas and remove the need for setting up automation for each strategy (e.g. using Chainlink keepers). The strategy could never spend more than the cumulative daily budgets. But it may lead to  sudden usage of a large amount of accumulated budget which may not be intended.


## Assessed type

Timing