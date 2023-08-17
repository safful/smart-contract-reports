## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-07

# [TWA update is not correct](https://github.com/code-423n4/2022-12-Stealth-Project-findings/issues/105) 

# Lines of code

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L294


# Vulnerability details

## Impact

Time-warped-price is updated incorrectly and this affects moving bins.

## Proof of Concept

The protocol updates `twa` on every swap and uses that to decide how to move bins.
But in the function `swap()`, the delta's `endSqrtPrice` can not contribute negatively to the `activeTick` because it is wrapped with a `clip()` function.

```solidity
// Pool.sol
286:         if (amountOut != 0) {
287:             if (tokenAIn) {
288:                 binBalanceA += delta.deltaInBinInternal.toUint128();
289:                 binBalanceB = Math.clip128(binBalanceB, delta.deltaOutErc.toUint128());
290:             } else {
291:                 binBalanceB += delta.deltaInBinInternal.toUint128();
292:                 binBalanceA = Math.clip128(binBalanceA, delta.deltaOutErc.toUint128());
293:             }
294:             twa.updateValue(currentState.activeTick * PRBMathSD59x18.SCALE + int256(Math.clip(delta.endSqrtPrice, delta.sqrtLowerTickPrice).div(delta.sqrtUpperTickPrice - delta.sqrtLowerTickPrice)));//@audit second part of the sum is always positive, should contribute both ways
295:         }
296:
```

I believe `delta.endSqrtPrice` should contribute both ways to the `activeTick` and it is quite possible for the `endSqrtPrice` to be out of the range `(delta.sqrtLowerTickPrice, delta.sqrtUpperTickPrice)`. (In another report, I mentioned an issue of accuracy loss in the calculation of `endSqrtPrice`).

## Tools Used

Manual Review

## Recommended Mitigation Steps

I recommend changing the relevant line as below without using `clip()` so that the `endSqrtPrice` can contribute to the `twa` reasonably.

```solidity
294:             twa.updateValue(currentState.activeTick * PRBMathSD59x18.SCALE + int256((delta.endSqrtPrice, delta.sqrtLowerTickPrice).div(delta.sqrtUpperTickPrice - delta.sqrtLowerTickPrice)));
```