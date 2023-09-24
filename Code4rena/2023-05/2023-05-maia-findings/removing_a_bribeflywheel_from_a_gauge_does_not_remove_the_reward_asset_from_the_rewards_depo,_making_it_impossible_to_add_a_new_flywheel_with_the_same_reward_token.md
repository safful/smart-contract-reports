## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-28

# [Removing a BribeFlywheel from a Gauge does not remove the reward asset from the rewards depo, making it impossible to add a new Flywheel with the same reward token](https://github.com/code-423n4/2023-05-maia-findings/issues/214) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L144-L152


# Vulnerability details

## Impact

Removing a bribe Flywheel (`FlywheelCore`) from a Gauge (via `BaseV2Gauge::removeBribeFlywheel`) does not remove the reward asset (call `MultiRewardsDepot::removeAsset`) from the rewards depo (`BaseV2Gauge::multiRewardsDepot`), making it impossible to add a new Flywheel (by calling `BaseV2Gauge::addBribeFlywheel`) with the same reward token (because `MultiRewardsDepot::addAsset` reverts as the assets already exists).

Impact is limiting protocol functionality in unwanted ways, possibly impacting gains in the long run. Example due to incentives lost by not having a specific token bribe reward.

## Proof of Concept

_Observation: a `BribeFlywheel` is a `FlywheelCore` with a `FlywheelBribeRewards` set as the `FlywheelRewards`, typically created using the `BribesFactory::createBribeFlywheel`_

### Scenario and execution flow

- project decides to add an initial  `BribeFlywheel` to the recently deployed `UniswapV3Gauge` contract.
- this is done by calling [`UniswapV3GaugeFactory::BaseV2GaugeFactory::addBribeToGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L144-L148)
    - executions further goes to `BaseV2Gauge::addGaugetoFlywheel` where [the bribe flywheel reward token is added](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L135) to the multi reward depo
- project decides, for whatever reason (a bug in the contract, an exploit, a decommission, a more profitable wheel that would use the same rewards token), that they want to replace the old flywheel with a new one
- removing is done via calling [`UniswapV3GaugeFactory::BaseV2GaugeFactory::removeBribeFromGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L151-L154)
    - executions further goes to `BaseV2Gauge::removeBribeFlywheel` where the flywheel is removed but the reward token asset [is not remove from the multi reward depo](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L144-L152), there is no call to [`MultiRewardsDepot::removeAsset`](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/depots/MultiRewardsDepot.sol#L57-L65):

```Solidity
    function removeBribeFlywheel(FlywheelCore bribeFlywheel) external onlyOwner {
        /// @dev Can only remove active flywheels
        if (!isActive[bribeFlywheel]) revert FlywheelNotActive();

        /// @dev This is permanent; can't be re-added
        delete isActive[bribeFlywheel];

        emit RemoveBribeFlywheel(bribeFlywheel);
    }
```
- after removal, when trying to add a new flywheel with the same rewards token, execution fails with `ErrorAddingAsset` since the [`addAsset`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L135) call reverts since [the rewards token was not removed](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/depots/MultiRewardsDepot.sol#L48) with the previous call to `BaseV2Gauge::removeBribeFlywheel`.

## Tools Used

Manual analysis

## Recommended Mitigation Steps

when `BaseV2Gauge::removeBribeFlywheel` is called for a particular flywheel, also remove it's corresponding reward depo token.

Example implementation
```diff
diff --git a/src/gauges/BaseV2Gauge.sol b/src/gauges/BaseV2Gauge.sol
index c2793a7..8ea6c1e 100644
--- a/src/gauges/BaseV2Gauge.sol
+++ b/src/gauges/BaseV2Gauge.sol
@@ -148,6 +148,9 @@ abstract contract BaseV2Gauge is Ownable, IBaseV2Gauge {
         /// @dev This is permanent; can't be re-added
         delete isActive[bribeFlywheel];
 
+        address flyWheelRewards = address(bribeFlywheel.flywheelRewards());        
+        multiRewardsDepot.removeAsset(flyWheelRewards);
+
         emit RemoveBribeFlywheel(bribeFlywheel);
     }
 

```


## Assessed type

Other