## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-09

# [_decrementWeightUntilFree() Possible infinite loop](https://github.com/code-423n4/2023-05-maia-findings/issues/735) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Gauges.sol#L547-L549


# Vulnerability details

## Impact
the position of `i++` is wrong, which may lead to an infinite loop

## Proof of Concept
In the loop of the `_decrementWeightUntilFree()` method, the position of `i++` is wrong, which may lead to a infinite loop

```solidity
    function _decrementWeightUntilFree(address user, uint256 weight) internal nonReentrant {
...
        for (uint256 i = 0; i < size && (userFreeWeight + totalFreed) < weight;) {
            address gauge = gaugeList[i];
            uint112 userGaugeWeight = getUserGaugeWeight[user][gauge];
            if (userGaugeWeight != 0) {
                // If the gauge is live (not deprecated), include its weight in the total to remove
                if (!_deprecatedGauges.contains(gauge)) {
                    totalFreed += userGaugeWeight;
                }
                userFreed += userGaugeWeight;
                _decrementGaugeWeight(user, gauge, userGaugeWeight, currentCycle);
                unchecked {
@>                  i++;
                }
            }
        }
```

In the above code, when `userGaugeWeight == 0`, `i` is not incremented, resulting in a infinite loop.
The current protocol does not restrict `getUserGaugeWeight[user][gauge] == 0`.

## Tools Used

## Recommended Mitigation Steps

```solidity
    function _decrementWeightUntilFree(address user, uint256 weight) internal nonReentrant {
...
        for (uint256 i = 0; i < size && (userFreeWeight + totalFreed) < weight;) {
            address gauge = gaugeList[i];
            uint112 userGaugeWeight = getUserGaugeWeight[user][gauge];
            if (userGaugeWeight != 0) {
                // If the gauge is live (not deprecated), include its weight in the total to remove
                if (!_deprecatedGauges.contains(gauge)) {
                    totalFreed += userGaugeWeight;
                }
                userFreed += userGaugeWeight;
                _decrementGaugeWeight(user, gauge, userGaugeWeight, currentCycle);
-               unchecked {
-                  i++;
-               }
            }
+           unchecked {
+             i++;
+           }            
        }
```



## Assessed type

Context