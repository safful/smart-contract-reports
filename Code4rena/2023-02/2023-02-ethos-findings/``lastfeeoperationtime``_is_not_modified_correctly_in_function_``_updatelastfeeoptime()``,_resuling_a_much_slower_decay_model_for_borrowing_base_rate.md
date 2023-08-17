## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-14

# [``lastFeeOperationTime`` is not modified correctly in function ``_updateLastFeeOpTime()``, resuling a much slower decay model for borrowing base rate](https://github.com/code-423n4/2023-02-ethos-findings/issues/33) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/TroveManager.sol#L1500-L1507


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
``lastFeeOperationTime`` is not modified correctly in function ``_updateLastFeeOpTime()``. As a result, ``decayBaseRateFromBorrowing()``  will decay the base rate more slowly than expected (worst case half slower). 

Since borrowing base rate is so fundamental to the protocol, I would rate this finding as H.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

We show how ``decayBaseRateFromBorrowing()``  will decay the base rate more slowly than expected because of the wrong modification of ``lastFeeOperationTime`` in ``_updateLastFeeOpTime()``:

1) ``decayBaseRateFromBorrowing()``  calls ``_calcDecayedBaseRate()`` to calculate the decayed base rate based how many minutes elapsed since last recorded ``lastFeeOperationTime``. 

```javascript
function decayBaseRateFromBorrowing() external override {
        _requireCallerIsBorrowerOperations();

        uint decayedBaseRate = _calcDecayedBaseRate();
        assert(decayedBaseRate <= DECIMAL_PRECISION);  // The baseRate can decay to 0

        baseRate = decayedBaseRate;
        emit BaseRateUpdated(decayedBaseRate);

        _updateLastFeeOpTime();
    }

   function _calcDecayedBaseRate() internal view returns (uint) {
        uint minutesPassed = _minutesPassedSinceLastFeeOp();
        uint decayFactor = LiquityMath._decPow(MINUTE_DECAY_FACTOR, minutesPassed);

        return baseRate.mul(decayFactor).div(DECIMAL_PRECISION);
    }

 function _minutesPassedSinceLastFeeOp() internal view returns (uint) {
        return (block.timestamp.sub(lastFeeOperationTime)).div(SECONDS_IN_ONE_MINUTE);
 }
```

2) ``decayBaseRateFromBorrowing()``  then calls ``_updateLastFeeOpTime()`` to set ``lastFeeOperationTime`` to the current time if at least 1 minute pass. 

```javascript
 function _updateLastFeeOpTime() internal {
        uint timePassed = block.timestamp.sub(lastFeeOperationTime);

        if (timePassed >= SECONDS_IN_ONE_MINUTE) {
            lastFeeOperationTime = block.timestamp;
            emit LastFeeOpTimeUpdated(block.timestamp);
        }
    }
```

3) The problem with such an update of ``lastFeeOperationTime`` is, if 1.999 minutes had passed, the base rate will only decay for 1 minute, at the same time, 1.999 minutes will be added on``lastFeeOperationTime``. In other words, in a worst scenario, for every 1.999 minutes, the base rate will only decay for 1 minute. Therefore, the base rate will decay more slowly then expected. 

4) The borrowing base rate is very fundamental to the whole protocol. Any small deviation is  accumulative. In the worse case, the decay speed will slow down by half; on average, it will be 0.75 slower. 

## Tools Used
VSCode

## Recommended Mitigation Steps
Using  the effective elapsed time that is consumed by the model so far to revise ``lastFeeOperationTime``.

```diff
 function _updateLastFeeOpTime() internal {
        uint timePassed = block.timestamp.sub(lastFeeOperationTime);

        if (timePassed >= SECONDS_IN_ONE_MINUTE) {
-            lastFeeOperationTime = block.timestamp;
+            lastFeeOperationTime += _minutesPassedSinceLastFeeOp()*SECONDS_IN_ONE_MINUTE;
            emit LastFeeOpTimeUpdated(block.timestamp);
        }
    }
```
