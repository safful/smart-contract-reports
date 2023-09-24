## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-26

# [If `HERMES` gauge rewards are not queued for distribution every week, they are slashed](https://github.com/code-423n4/2023-05-maia-findings/issues/411) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L78-L80


# Vulnerability details

## Impact

In order to queue weekly `HERMES` rewards for distribution, `FlywheelGaugeRewards::queueRewardsForCycle` must be called during the next cycle (week). If a cycle has passed and no one calls `queueRewardsForCycle` to queue rewards, cycle gauge rewards are lost as internal accounting does not take into consideration time passing, only last processed cycle.

### Issue details

The minter kicks off a new epoch via calling `BaseV2Minter::updatePeriod`. The execution flow goes to `FlywheelGaugeRewards::queueRewardsForCycle` -> `FlywheelGaugeRewards::_queueRewards` where after several checks the [rewards are queued](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L189-L193) in order for them to be retrieved via a call to [`FlywheelGaugeRewards::getAccruedRewards`](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L200) from [`BaseV2Gauge::newEpoch`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L89).

Reward queuing logic revolves around the [current and previously saved gauge cycle](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L78-L80):

```Solidity
        // next cycle is always the next even divisor of the cycle length above current block timestamp.
        uint32 currentCycle = (block.timestamp.toUint32() / gaugeCycleLength) * gaugeCycleLength;
        uint32 lastCycle = gaugeCycle;
```

This way of noting cycles ([and further checks done](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L82-L85)) does not take into consideration any intermediary cycles, only that new cycle is after old cycle. If `queueRewardsForCycle` is not called for a number of cycles then rewards will be lost for those cycles.

## Proof of Concept

Rewards are calculate for current cycle and last stored cycle only, with no intermediary accounting:
https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L78-L80

Visual example:
```
 0 1 2 3 4 5 6  (epoch/cycle)
+-+-+-+-+-+-+-+
|Q|Q|Q| | |Q|Q|
+-+-+-+-+-+-+-+
```

Up until epoch 2 `queueRewardsForCycle` (Q) was called, for cycle 3 and 4 nobody calls, on cycle 5 `queueRewardsForCycle` is called again but cycle 3 and 4 rewards are not taken into consideration.


## Tools Used

Manual analysis

## Recommended Mitigation Steps

Because of the way the entire MaiaDAO ecosystem is set up, the premise is that someone will call `BaseV2Minter::updatePeriod` (which calls `FlywheelGaugeRewards::queueRewardsForCycle`) as there is incentive for users (or project) to do so. Realistically this _should_ always happen, but unforeseen events may lead to this event.

It is difficult from an architectural point of view, regarding how MaiaDAO is constructed, to offer a solution. A generic suggestion would be to implemented a snapshot mechanism / dynamic accounting of each cycle but then the issue would be who triggers that snapshot event? 

This issue is real but mitigating it is not straightforward or evident in web3 context. One workaround is to use proper on-chain automation such as [Chainlink Automation](https://docs.chain.link/chainlink-automation/introduction).







## Assessed type

Other