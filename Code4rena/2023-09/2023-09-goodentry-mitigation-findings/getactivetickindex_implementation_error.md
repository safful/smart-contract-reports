## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- MR-M-03

# [getActiveTickIndex implementation error](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/43) 

# Lines of code

https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/GeVault.sol#L470


# Vulnerability details

## Impact

The implementation of getActiveTickIndex is wrong, and the searched ticks do not meet expectations, causing funds to be incorrectly allocated to edge ticks, and there is basically no staking income.

## Proof of Concept

```solidity
    // if base token is token0, ticks above only contain base token = token0 and ticks below only hold quote token = token1
    if (newTickIndex > 1) 
      depositAndStash(
        ticks[newTickIndex-2], 
        baseTokenIsToken0 ? 0 : availToken0 / liquidityPerTick,
        baseTokenIsToken0 ? availToken1 / liquidityPerTick : 0
      );


  /// @notice Return first valid tick
  function getActiveTickIndex() public view returns (uint activeTickIndex) {
    // loop on all ticks, if underlying is only base token then we are above, and tickIndex is 2 below
    for (uint tickIndex = 0; tickIndex < ticks.length; tickIndex++){
      (uint amt0, uint amt1) = ticks[tickIndex].getTokenAmountsExcludingFees(1e18);
      // found a tick that's above price (ie its only underlying is the base token)
      if( (baseTokenIsToken0 && amt0 == 0) || (!baseTokenIsToken0 && amt0 == 0) ) return tickIndex;
    }
    // all ticks are below price
    return ticks.length;
  }
```

According to code comments:
- If baseTokenIsToken0 is true, ticks above current price only contain base token, that is token0, so amt1 is 0.
- And if baseTokenIsToken0 is false, ticks below current price only contain quote token, that is token1, so amt0 is 0.

getActiveTickIndex checks amt0 twice in the code is wrong, which causes `baseTokenIsToken0 && amt0 == 0` to be true when the tick is below the current price.
That is, the searched tick is the first tick lower than the current price, not the first tick greater than the current price,  which is the first tick in the list. 
This results in funds being staked to marginal ticks and unable to obtain staking income.


## Tools Used

Manual review

## Recommended Mitigation Steps

```solidity
      // found a tick that's above price (ie its only underlying is the base token)
      if( (baseTokenIsToken0 && amt1 == 0) || (!baseTokenIsToken0 && amt0 == 0) ) return tickIndex;
```


## Assessed type

Context