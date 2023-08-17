## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-10

# [P can be updated to zero which can cause a DOS when liquidating troves](https://github.com/code-423n4/2023-02-ethos-findings/issues/338) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/StabilityPool.sol#L575-L595


# Vulnerability details

## Impact
P is asserted to never be zero in `_updateRewardSumAndProduct` which is called for every liquidation. However, there is an edge case that can cause P to be zero, causing DOS to certain liquidations.

## Proof of Concept

In `_updateRewardSumAndProduct` notice `assert(newP > 0);`. However, the SCALE_FACTOR check is insufficient in ensuring P is always more than zero. Three cycles of very small P can cause P to drop to zero and hence causing revert for that particular liquidation of troves. POC steps are as follows,

1. First liquidation newProductFactor is 1e2, 1e18*1e2/1e18 = 1e2,  since its less than scale factor. 1e9 is multiplied before dividing with 1e18. P = 1e11 after 1st liquidation cycle.
2. Second liquidation newProductFactor is 1e3, 1e11*1e3/1e18 = 0, since its less than scale factor. 1e9 is multipled. P = 1e5
3. Third liquidation newProductFactor is 1e3, 1e5*1e3/1e8 = 0, since its less than scale factor, 1e9 is multipled. However, even after 1e9 is multiplied it will result in 1e17, which is less than DECIMAL_PRECISION of 1e18. P = 0. 

4. Revert due to `assert(newP > 0);`.


```solidity
        // If the Stability Pool was emptied, increment the epoch, and reset the scale and product P
        if (newProductFactor == 0) {
            currentEpoch = currentEpochCached.add(1);
            emit EpochUpdated(currentEpoch);
            currentScale = 0;
            emit ScaleUpdated(currentScale);
            newP = DECIMAL_PRECISION;


        // If multiplying P by a non-zero product factor would reduce P below the scale boundary, increment the scale
        } else if (currentP.mul(newProductFactor).div(DECIMAL_PRECISION) < SCALE_FACTOR) {
            newP = currentP.mul(newProductFactor).mul(SCALE_FACTOR).div(DECIMAL_PRECISION); 
            currentScale = currentScaleCached.add(1);
            emit ScaleUpdated(currentScale);
        } else {
            newP = currentP.mul(newProductFactor).div(DECIMAL_PRECISION);
        }


        assert(newP > 0);
        P = newP;


        emit P_Updated(newP);
    }
```

Reasons for putting high is because

1. It breaks the invariant of P being > 0 which according to the docs would break deposit tracking when Pool is not empty.

2. Even though in some lucky cases, smaller liquidation can be made if batch liquidation is done, in the event that P is at a precarious place whereby all permutations of liquidations done will result in P being 0, liquidation will be DOSed which can be detrimental to protocol as they cannot liquidate bad debt and hence might lead it to insolvency.  

## Tools Used

Manual Review

## Recommended Mitigation Steps

Recommend setting SCALE_FACTOR to 1e18, even though the docs did explain that 1e9 is used instead of 1e18 to ensure negligible precision loss, the alternative option is redesigning the mitigation mechanism of rounding error for P.



