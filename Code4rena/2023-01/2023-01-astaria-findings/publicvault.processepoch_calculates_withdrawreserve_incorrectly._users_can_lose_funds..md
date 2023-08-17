## Tags

- bug
- 3 (High Risk)
- grade-a
- satisfactory
- selected for report
- H-18

# [PublicVault.processEpoch calculates withdrawReserve incorrectly. Users can lose funds.](https://github.com/code-423n4/2023-01-astaria-findings/issues/157) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L275-L343


# Vulnerability details

## Impact
PublicVault.processEpoch calculates withdrawReserve incorrectly. As result user can receive less funds when totalAssets() <= expected from auction.
## Proof of Concept
When users wants to withdraw from `PublicVault` then `WithdrawProxy` is deployed and `PublicVault.processEpoch` function is responsible to calculate `s.withdrawReserve`.
This amount depends on how many shares should be redeemed and if there is auction for the epoch.

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L275-L343
```
solidity
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


    // increment epoch
    unchecked {
      s.currentEpoch++;
    }
  }
```

` s.liquidationWithdrawRatio` depends on how many shares exists inside WithdrawProxy. In case if amount of shares inside `WithdrawProxy` equal to amount of shares inside `PublicVault` that means that withdraw ratio is 100% and all funds from Vault should be sent to `WithdrawProxy`.

In case if auction is in progress then `WithdrawProxy.getExpected` is not 0 and some amount of funds is expected from auction.
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
```
This is `s.withdrawReserve` calculation. As you can see in case if `totalAssets() <= expected` then `s.withdrawReserve` is set to 0 and no funds will be sent to proxy. This is incorrect though.

For example in the case when withdraw ratio is 100% all funds should be sent to the withdraw proxy, but because of that check, some part of funds will be still inside the vault and depositors will lose their funds. If for example totalAssets is 5eth and expected is 5 eth, then depositors will lose all 5 eth.

This check is done in such way, because of [calculations inside `WithdrawProxy`](https://github.com/code-423n4/2023-01-astaria/blob/main/src/WithdrawProxy.sol#L258-L266). But it's not correct.
## Tools Used
VsCode
## Recommended Mitigation Steps
You need to check this logic again. Maybe you need to always send `s.withdrawReserve = totalAssets().mulWadDown(s.liquidationWithdrawRatio).safeCastTo88()` amount to the withdraw proxy. But then you need rethink, how WithdrawProxy will handle yIntercept increase/decrease.