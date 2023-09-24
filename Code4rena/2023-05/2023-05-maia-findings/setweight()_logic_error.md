## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-03

# [setWeight() Logic error](https://github.com/code-423n4/2023-05-maia-findings/issues/766) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesPool.sol#L223


# Vulnerability details

## Impact
Logic error

## Proof of Concept
`setWeight()`Used to set the new weight
The code is as follows:
```solidity
    function setWeight(uint256 poolId, uint8 weight) external nonReentrant onlyOwner {
        if (weight == 0) revert InvalidWeight();

        uint256 poolIndex = destinations[poolId];

        if (poolIndex == 0) revert NotUlyssesLP();

        uint256 oldRebalancingFee;

        for (uint256 i = 1; i < bandwidthStateList.length; i++) {
            uint256 targetBandwidth = totalSupply.mulDiv(bandwidthStateList[i].weight, totalWeights);

            oldRebalancingFee += _calculateRebalancingFee(bandwidthStateList[i].bandwidth, targetBandwidth, false);
        }

        uint256 oldTotalWeights = totalWeights;
        uint256 weightsWithoutPool = oldTotalWeights - bandwidthStateList[poolIndex].weight;
        uint256 newTotalWeights = weightsWithoutPool + weight;
        totalWeights = newTotalWeights;

        if (totalWeights > MAX_TOTAL_WEIGHT || oldTotalWeights == newTotalWeights) {
            revert InvalidWeight();
        }

        uint256 leftOverBandwidth;

        BandwidthState storage poolState = bandwidthStateList[poolIndex];
        poolState.weight = weight;
@>      if (oldTotalWeights > newTotalWeights) {
            for (uint256 i = 1; i < bandwidthStateList.length;) {
                if (i != poolIndex) {
                    uint256 oldBandwidth = bandwidthStateList[i].bandwidth;
                    if (oldBandwidth > 0) {
                        bandwidthStateList[i].bandwidth =
                            oldBandwidth.mulDivUp(oldTotalWeights, newTotalWeights).toUint248();
                        leftOverBandwidth += oldBandwidth - bandwidthStateList[i].bandwidth;
                    }
                }

                unchecked {
                    ++i;
                }
            }
            poolState.bandwidth += leftOverBandwidth.toUint248();
        } else {
            uint256 oldBandwidth = poolState.bandwidth;
            if (oldBandwidth > 0) {
@>              poolState.bandwidth = oldBandwidth.mulDivUp(oldTotalWeights, newTotalWeights).toUint248();

                leftOverBandwidth += oldBandwidth - poolState.bandwidth;
            }

            for (uint256 i = 1; i < bandwidthStateList.length;) {
              

                if (i != poolIndex) {
                     if (i == bandwidthStateList.length - 1) {
@>                       bandwidthStateList[i].bandwidth += leftOverBandwidth.toUint248();
                     } else if (leftOverBandwidth > 0) {
@>                       bandwidthStateList[i].bandwidth +=
@>                          leftOverBandwidth.mulDiv(bandwidthStateList[i].weight, weightsWithoutPool).toUint248();
                     }
                }

                unchecked {
                    ++i;
                }
            }
        }
```

There are several problems with the above code
1. `if (oldTotalWeights > newTotalWeights)` should be changed to `if (oldTotalWeights < newTotalWeights)` because the logic inside the if is to calculate the case of increasing `weight`

2. `poolState.bandwidth = oldBandwidth.mulDivUp(oldTotalWeights , newTotalWeights).toUint248();`
should be modified to `poolState.bandwidth = oldBandwidth.mulDivUp(newTotalWeights, oldTotalWeights).toUint248();`
Because this calculates the extra number

3. `leftOverBandwidth` has a problem with the processing logic



## Tools Used

## Recommended Mitigation Steps

```solidity
    function setWeight(uint256 poolId, uint8 weight) external nonReentrant onlyOwner {
...

-        if (oldTotalWeights > newTotalWeights) {
+        if (oldTotalWeights < newTotalWeights) {
            for (uint256 i = 1; i < bandwidthStateList.length;) {
                if (i != poolIndex) {
                    uint256 oldBandwidth = bandwidthStateList[i].bandwidth;
                    if (oldBandwidth > 0) {
                        bandwidthStateList[i].bandwidth =
                            oldBandwidth.mulDivUp(oldTotalWeights, newTotalWeights).toUint248();
                        leftOverBandwidth += oldBandwidth - bandwidthStateList[i].bandwidth;
                    }
                }

                unchecked {
                    ++i;
                }
            }
            poolState.bandwidth += leftOverBandwidth.toUint248();
        } else {
            uint256 oldBandwidth = poolState.bandwidth;
            if (oldBandwidth > 0) {
-               poolState.bandwidth = oldBandwidth.mulDivUp(oldTotalWeights, newTotalWeights).toUint248();
+               poolState.bandwidth = oldBandwidth.mulDivUp(newTotalWeights, oldTotalWeights).toUint248();

                leftOverBandwidth += oldBandwidth - poolState.bandwidth;
            }

+           uint256 currentGiveWidth = 0;
+           uint256 currentGiveCount = 0;
            for (uint256 i = 1; i < bandwidthStateList.length;) {

+                if (i != poolIndex) {
+                     if(currentGiveCount == bandwidthStateList.length - 2 - 1) { //last
+                         bandwidthStateList[i].bandwidth += leftOverBandwidth - currentGiveWidth;
+                    }
+                     uint256 sharesWidth = leftOverBandwidth.mulDiv(bandwidthStateList[i].weight, weightsWithoutPool).toUint248();
+                     bandwidthStateList[i].bandwidth += sharesWidth;
+                     currentGiveWidth +=sharesWidth;  
+                     currentCount++;
+                 }                

-                if (i != poolIndex) {
-                    if (i == bandwidthStateList.length - 1) {
-                        bandwidthStateList[i].bandwidth += leftOverBandwidth.toUint248();
-                    } else if (leftOverBandwidth > 0) {
-                        bandwidthStateList[i].bandwidth +=
-                            leftOverBandwidth.mulDiv(bandwidthStateList[i].weight, weightsWithoutPool).toUint248();
-                    }
-               }

                unchecked {
                    ++i;
                }
            }
        }
...

```


## Assessed type

Context