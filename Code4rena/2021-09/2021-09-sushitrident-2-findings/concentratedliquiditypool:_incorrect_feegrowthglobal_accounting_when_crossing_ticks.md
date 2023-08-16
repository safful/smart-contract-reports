## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [ConcentratedLiquidityPool: incorrect feeGrowthGlobal accounting when crossing ticks](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/16) 

# Handle

hickuphh3


# Vulnerability details

### Impact

Swap fees are taken from the output. Hence, if swapping token0 for token1 (`zeroForOne` is true), then fees are taken in token1. We see this to be the case in the initialization of feeGrowthGlobal in the swap cache

 `feeGrowthGlobal = zeroForOne ? feeGrowthGlobal1 : feeGrowthGlobal0;`

and in `_updateFees()`.

However, looking at `Ticks.cross()`, the logic is the reverse, which causes wrong fee accounting.

```jsx
if (zeroForOne) {
	...
	ticks[nextTickToCross].feeGrowthOutside0 = feeGrowthGlobal - ticks[nextTickToCross].feeGrowthOutside0;
} else {
	...
	ticks[nextTickToCross].feeGrowthOutside1 = feeGrowthGlobal - ticks[nextTickToCross].feeGrowthOutside1;
}
```

### Recommended Mitigation Steps

Switch the `0` and `1` in `Ticks.cross()`.

```jsx
if (zeroForOne) {
	...
	// feeGrowthGlobal = feeGrowthGlobal1
	ticks[nextTickToCross].feeGrowthOutside1 = feeGrowthGlobal - ticks[nextTickToCross].feeGrowthOutside1;
} else {
	...
	// feeGrowthGlobal = feeGrowthGlobal0
	ticks[nextTickToCross].feeGrowthOutside0 = feeGrowthGlobal - ticks[nextTickToCross].feeGrowthOutside0;
}
```

