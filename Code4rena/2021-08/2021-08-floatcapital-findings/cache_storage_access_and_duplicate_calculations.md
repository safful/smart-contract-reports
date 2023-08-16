## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- fixed-in-upstream-repo

# [Cache storage access and duplicate calculations](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/106) 

# Handle

pauliax


# Vulnerability details

## Impact
functions _mintNextPrice, _redeemNextPrice, _shiftPositionNextPrice could cache a result here and re-use it to avoid duplicate calculations of the same value:
    marketUpdateIndex[marketIndex] + 1;
Also, you can extract duplicate storage access to a storage variable and update the state on it, e.g.: accumulativeFloatPerSyntheticTokenSnapshots[marketIndex][newIndex] is accessed 3 times in _setCurrentAccumulativeIssuancePerStakeStakedSynthSnapshot.

