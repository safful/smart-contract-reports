## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- M-29

# [PublicVault.processEpoch updates YIntercept incorrectly when totalAssets() <= expected](https://github.com/code-423n4/2023-01-astaria-findings/issues/124) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L275-L337


# Vulnerability details

## Impact
PublicVault.processEpoch updates YIntercept incorrectly when totalAssets() <= expected.

## Proof of Concept
When `processEpoch` is called it calculates amount of `withdrawReserve` that will be sent to the withdraw proxy.
Later it updates `yIntercept` variable.
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L275-L337
```solidity
  function processEpoch() public {
    // check to make sure epoch is over
    if (timeToEpochEnd() > 0) {
      revert InvalidState(InvalidStates.EPOCH_NOT_OVER);
    }
    VaultData storage s = _loadStorageSlot();


    if (s.withdrawReserve > 0) {
      revert InvalidState(InvalidStates.WITHDRAW_RESERVE_NOT_ZERO);
    }


    WithdrawProxy currentWithdrawProxy = WithdrawProxy(
      s.epochData[s.currentEpoch].withdrawProxy
    );


    // split funds from previous WithdrawProxy with PublicVault if hasn't been already
    if (s.currentEpoch != 0) {
      WithdrawProxy previousWithdrawProxy = WithdrawProxy(
        s.epochData[s.currentEpoch - 1].withdrawProxy
      );
      if (
        address(previousWithdrawProxy) != address(0) &&
        previousWithdrawProxy.getFinalAuctionEnd() != 0
      ) {
        previousWithdrawProxy.claim();
      }
    }


    if (s.epochData[s.currentEpoch].liensOpenForEpoch > 0) {
      revert InvalidState(InvalidStates.LIENS_OPEN_FOR_EPOCH_NOT_ZERO);
    }


    // reset liquidationWithdrawRatio to prepare for re calcualtion
    s.liquidationWithdrawRatio = 0;


    // check if there are LPs withdrawing this epoch
    if ((address(currentWithdrawProxy) != address(0))) {
      uint256 proxySupply = currentWithdrawProxy.totalSupply();


      s.liquidationWithdrawRatio = proxySupply
        .mulDivDown(1e18, totalSupply())
        .safeCastTo88();


      currentWithdrawProxy.setWithdrawRatio(s.liquidationWithdrawRatio);
      uint256 expected = currentWithdrawProxy.getExpected();


      unchecked {
        if (totalAssets() > expected) {
          s.withdrawReserve = (totalAssets() - expected)
            .mulWadDown(s.liquidationWithdrawRatio)
            .safeCastTo88();
        } else {
          s.withdrawReserve = 0;
        }
      }
      _setYIntercept(
        s,
        s.yIntercept -
          totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)
      );
      // burn the tokens of the LPs withdrawing
      _burn(address(this), proxySupply);
    }
```

The part that we need to investigate is this.
```solidity
unchecked {
        if (totalAssets() > expected) {
          s.withdrawReserve = (totalAssets() - expected)
            .mulWadDown(s.liquidationWithdrawRatio)
            .safeCastTo88();
        } else {
          s.withdrawReserve = 0;
        }
      }
      _setYIntercept(
        s,
        s.yIntercept -
          totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)
      );
```

In case if `totalAssets() > expected` then `withdrawReserve`  is `totalAssets() - expected` multiplied by `liquidationWithdrawRatio`.
That means that `withdrawReserve` amount will be sent of public vault to the withdraw proxy, so total assets should decrease by this amount.
In this case call of `_setYIntercept` bellow is correct.

However in case when `totalAssets() <= expected` then `withdrawReserve` is set to 0, that means that nothing will be sent to the withdraw proxy. But `_setYIntercept` is still called in this case and total assets is decreased, but should not.

## Tools Used
VsCode
## Recommended Mitigation Steps
In case when `totalAssets() <= expected` do not call `_setYIntercept`.