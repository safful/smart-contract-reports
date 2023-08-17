## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-32

# [Certain function can be blocked if the ERC20 token revert in 0 amount transfer after PublicVault#transferWithdrawReserve is called](https://github.com/code-423n4/2023-01-astaria-findings/issues/54) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L295
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L421
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L359
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L372
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L384


# Vulnerability details

## Impact 

Certain function can be blocked if the ERC20 token revert in 0 amount transfer after PublicVault#transferWithdrawReserve is called

## Proof of Concept

The function transferWithdrawReserve in Public Vault has no access control.

```solidity
  function transferWithdrawReserve() public {
    VaultData storage s = _loadStorageSlot();

    if (s.currentEpoch == uint64(0)) {
      return;
    }

    address currentWithdrawProxy = s
      .epochData[s.currentEpoch - 1]
      .withdrawProxy;
    // prevents transfer to a non-existent WithdrawProxy
    // withdrawProxies are indexed by the epoch where they're deployed
    if (currentWithdrawProxy != address(0)) {
      uint256 withdrawBalance = ERC20(asset()).balanceOf(address(this));

      // prevent transfer of more assets then are available
      if (s.withdrawReserve <= withdrawBalance) {
        withdrawBalance = s.withdrawReserve;
        s.withdrawReserve = 0;
      } else {
        unchecked {
          s.withdrawReserve -= withdrawBalance.safeCastTo88();
        }
      }

      ERC20(asset()).safeTransfer(currentWithdrawProxy, withdrawBalance);
      WithdrawProxy(currentWithdrawProxy).increaseWithdrawReserveReceived(
        withdrawBalance
      );
      emit WithdrawReserveTransferred(withdrawBalance);
    }

    address withdrawProxy = s.epochData[s.currentEpoch].withdrawProxy;
    if (
      s.withdrawReserve > 0 &&
      timeToEpochEnd() == 0 &&
      withdrawProxy != address(0)
    ) {
      address currentWithdrawProxy = s
        .epochData[s.currentEpoch - 1]
        .withdrawProxy;
      uint256 drainBalance = WithdrawProxy(withdrawProxy).drain(
        s.withdrawReserve,
        s.epochData[s.currentEpoch - 1].withdrawProxy
      );
      unchecked {
        s.withdrawReserve -= drainBalance.safeCastTo88();
      }
      WithdrawProxy(currentWithdrawProxy).increaseWithdrawReserveReceived(
        drainBalance
      );
    }
  }
```

If this function is called, the token balance is transfered to withdrawProxy

```solidity
uint256 withdrawBalance = ERC20(asset()).balanceOf(address(this));
```

and

```solidity
ERC20(asset()).safeTransfer(currentWithdrawProxy, withdrawBalance);
```

However, according to 

https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers

Some tokens (e.g. LEND) revert when transfering a zero value amount.

If ERC20(asset()).balanceOf(address(this)) return 0, the transfer revert.

The impact is that transferWithdrawReserve is also used in the other place:

```solidity
  function commitToLien(
    IAstariaRouter.Commitment calldata params,
    address receiver
  )
    external
    whenNotPaused
    returns (uint256 lienId, ILienToken.Stack[] memory stack, uint256 payout)
  {
    _beforeCommitToLien(params);
    uint256 slopeAddition;
    (lienId, stack, slopeAddition, payout) = _requestLienAndIssuePayout(
      params,
      receiver
    );
    _afterCommitToLien(
      stack[stack.length - 1].point.end,
      lienId,
      slopeAddition
    );
  }
```

which calls:

```solidity
_beforeCommitToLien(params);
```

which calls:

```solidity
  function _beforeCommitToLien(IAstariaRouter.Commitment calldata params)
    internal
    virtual
    override(VaultImplementation)
  {
    VaultData storage s = _loadStorageSlot();

    if (s.withdrawReserve > uint256(0)) {
      transferWithdrawReserve();
    }
    if (timeToEpochEnd() == uint256(0)) {
      processEpoch();
    }
  }
```

which calls transferWithdrawReserve() which revet in 0 amount transfer.

Consider the case below:

1.  User A calls commitToLien transaction is pending in mempool.
2.  User B front-run User A's transaction by calling  transferWithdrawReserve() and the PublicVault has no ERC20 token balance or User B just want to call  transferWithdrawReserve and not try to front-run user A, but the impact and result is the same.
3.  User B's transaction executes first,
4.  User A first his transaction revert because the ERC20 token asset revert in 0 amount transfer in transferWithdrawReserve() call

```solidity
uint256 withdrawBalance = ERC20(asset()).balanceOf(address(this));

// prevent transfer of more assets then are available
if (s.withdrawReserve <= withdrawBalance) {
withdrawBalance = s.withdrawReserve;
s.withdrawReserve = 0;
} else {
unchecked {
  s.withdrawReserve -= withdrawBalance.safeCastTo88();
}
}

ERC20(asset()).safeTransfer(currentWithdrawProxy, withdrawBalance);
```

This revertion not only impact commitToLien, but also impact PublicVault.sol#updateVaultAfterLiquidation

```solidity
function updateVaultAfterLiquidation(
uint256 maxAuctionWindow,
AfterLiquidationParams calldata params
) public onlyLienToken returns (address withdrawProxyIfNearBoundary) {
VaultData storage s = _loadStorageSlot();

_accrue(s);
unchecked {
  _setSlope(s, s.slope - params.lienSlope.safeCastTo48());
}

if (s.currentEpoch != 0) {
  transferWithdrawReserve();
}
uint64 lienEpoch = getLienEpoch(params.lienEnd);
_decreaseEpochLienCount(s, lienEpoch);

uint256 timeToEnd = timeToEpochEnd(lienEpoch);
if (timeToEnd < maxAuctionWindow) {
  _deployWithdrawProxyIfNotDeployed(s, lienEpoch);
  withdrawProxyIfNearBoundary = s.epochData[lienEpoch].withdrawProxy;

  WithdrawProxy(withdrawProxyIfNearBoundary).handleNewLiquidation(
	params.newAmount,
	maxAuctionWindow
  );
}
```

transaction can revert in above code when calling

```solidity
if (s.currentEpoch != 0) {
  transferWithdrawReserve();
}
uint64 lienEpoch = getLienEpoch(params.lienEnd);
```

if the address has no ERC20 token balance and the ERC20 token revert in 0 amount transfer after PublicVault#transferWithdrawReserve is called first

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol just return and do nothing when  PublicVault#transferWithdrawReserve is called if the address has no ERC20 token balance.