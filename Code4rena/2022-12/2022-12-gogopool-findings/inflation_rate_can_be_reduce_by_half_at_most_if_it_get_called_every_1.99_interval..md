## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- fix security (sponsor)
- M-06

# [Inflation rate can be reduce by half at most if it get called every 1.99 interval.](https://github.com/code-423n4/2022-12-gogopool-findings/issues/648) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/RewardsPool.sol#L98


# Vulnerability details

## Impact
When doing inflation, function `getInflationAmt()` calculated number of intervals elapsed by dividing the duration with interval length. 
```solidity
function getInflationIntervalsElapsed() public view returns (uint256) {
    ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
    uint256 startTime = getInflationIntervalStartTime();
    if (startTime == 0) {
        revert ContractHasNotBeenInitialized();
    }
    return (block.timestamp - startTime) / dao.getInflationIntervalSeconds();
}
```

As we can noticed that, this calculation is rounding down, it means if `block.timestamp - startTime = 1.99 intervals`, it only account for `1 interval`.

However, when updating start time after inflating, it still update to current timestamp while it should only increased by `intervalLength * intervalsElapsed` instead.
```solidity
addUint(keccak256("RewardsPool.InflationIntervalStartTime"), inflationIntervalElapsedSeconds); 
// @audit should only update to oldStartTime + inverval * numInterval
setUint(keccak256("RewardsPool.RewardsCycleTotalAmt"), newTokens);
```

Since default value of inflation interval = 1 days and reward cycle length = 14 days, so the impact is reduced. However, these configs can be changed in the future.

## Proof of Concept
Consider the scenario: 
1. Assume last inflation time is `InflationIntervalStartTime = 100`. `InflationIntervalSeconds = 50`.
2. At `timestamp = 199`, function `getInflationAmt()` will calculate
```solidity
inflationIntervalsElapsed = (199 - 100) / 50 = 1
// Compute inflation for total inflation intervals elapsed
for (uint256 i = 0; i < inflationIntervalsElapsed; i++) {
    newTotalSupply = newTotalSupply.mulWadDown(inflationRate);
} // @audit only loop once.
```
3. And then in `inflate()` function, `InflationIntervalStartTime` is still updated to current timestamp, so `InflationIntervalStartTime = 199`.
4. If this sequence of actions are repeatedly used, we can easily see
```solidity
InflationIntervalStartTime = 199, inflated count = 1
InflationIntervalStartTime = 298, inflated count = 2
InflationIntervalStartTime = 397, inflated count = 3
InflationIntervalStartTime = 496, inflated count = 4
InflationIntervalStartTime = 595, inflated count = 5
```

While at `timestamp = 595`, inflated times should be 
`(595 - 100) / 50 = 9` instead.



## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider only increasing `InflationIntervalStartTime` by the amount of intervals time interval length.
