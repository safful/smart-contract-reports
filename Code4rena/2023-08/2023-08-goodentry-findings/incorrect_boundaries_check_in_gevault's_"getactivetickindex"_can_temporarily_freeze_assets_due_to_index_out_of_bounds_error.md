## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-03

# [Incorrect boundaries check in GeVault's "getActiveTickIndex" can temporarily freeze assets due to Index out of bounds error](https://github.com/code-423n4/2023-08-goodentry-findings/issues/379) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L431
https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/GeVault.sol#L357-L358


# Vulnerability details

`GeVault` stores the `TokenisableRange` instances it operates on in an ordered array:
```Solidity
  TokenisableRange[] public ticks;
```
Whenever a rebalancing happens, the `GeVault` contract withdraws all its liquidity and redeploys it on at most four ticks, starting from (and including) the one identified by the `getActiveTickIndex` function:
```Solidity
  function deployAssets() internal { 
    uint newTickIndex = getActiveTickIndex();
    // [...]
    uint tick0Index = newTickIndex;
    uint tick1Index = newTickIndex + 2;
    // [...] 
      depositAndStash(ticks[tick0Index], availToken0 / 2, 0);
      depositAndStash(ticks[tick0Index+1], availToken0 / 2, 0);
    // [...]
      depositAndStash(ticks[tick1Index], 0, availToken1 / 2);
      depositAndStash(ticks[tick1Index+1], 0, availToken1 / 2);
```

However, the `getActiveTickIndex()`, given that the termination condition of its `for` loop, can return indices up to `ticks.length - 3`, included (**because the increment of `activeTickIndex` that made the boundary check fail is kept**):
```Solidity
  /// @notice Return first valid tick
  function getActiveTickIndex() public view returns (uint activeTickIndex) {
    if (ticks.length >= 5){
      // looking for index at which the underlying asset differs from the next tick
      for (activeTickIndex = 0; activeTickIndex < ticks.length - 3; activeTickIndex++){
        // [...]
        if ( /* ... */ )
          break;
      }
    }
  }
```
So the highest value it can possibly return is `ticks.length - 3`. If we take this value and project where `rebalance()` will deploy assets, we'll have the ticks at the four indices: `[ticks.length - 3, ticks.length - 2, ticks.length - 1, ticks.length]`, and the last value will overflow, causing the rebalancing, and the liquidity operation that triggered it (if any) to fail. 

## Impact
Whenever the market is such that the `getActiveTickIndex` returns the last possible index, the contract will revert on any `rebalance`, `deposit`, and more importantly `withdraw` operations. Despite the impact including locking assets, this finding is reported as medium severity because the protocol governance could resolve the situation by adding extra ticks & rescue the assets without requiring a contract code upgrade.

## Proof of Concept
I have a running PoC in my environment, which I will keep aside and will be happy to provide if requested, but I would rather not share it because it's a monstrous setup with a couple of workarounds to not make it even worse.

High level my setup is:
- set up a GeVault with 6 ranges, most of which are mostly at higher prices than the market:
  - range at index 0 at 3001-3250 (implicit, e10)
  - 1 at 2751-3000
  - 2 at 2501-2750
  - 3 at 2251-2500
  - 4 at 2001-2250
  - 5 at 1751-2000
  - with USDC/WETH at 1870 (I forked mainnet at block 17811921)
  - call `getActiveTickIndex()` which will return `3` (higher than 2, which is the max value that would not make the fund deployment go out of range)
  - deposit some WETH - this will revert out of bounds

## Tools Used
Code review, IDE, foundry

## Recommended Mitigation Steps
Decrease by 1 the loop boundaries:
```Solidity
  /// @notice Return first valid tick
  function getActiveTickIndex() public view returns (uint activeTickIndex) {
    if (ticks.length >= 5){
      // looking for index at which the underlying asset differs from the next tick
-      for (activeTickIndex = 0; activeTickIndex < ticks.length - 3; activeTickIndex++){
+      for (activeTickIndex = 0; activeTickIndex < ticks.length - 4; activeTickIndex++){

        (uint amt0, uint amt1) = ticks[activeTickIndex+1].getTokenAmountsExcludingFees(1e18);
        (uint amt0n, uint amt1n) = ticks[activeTickIndex+2].getTokenAmountsExcludingFees(1e18);
        if ( (amt0 == 0 && amt0n > 0) || (amt1 == 0 && amt1n > 0) )
          break;
      }
    }
  }

```
Or use a dedicated loop variable, and explicitly return the correct value.





## Assessed type

Loop