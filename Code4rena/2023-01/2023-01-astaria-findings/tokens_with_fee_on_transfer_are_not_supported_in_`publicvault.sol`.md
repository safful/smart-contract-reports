## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-09

# [Tokens with fee on transfer are not supported in `PublicVault.sol`](https://github.com/code-423n4/2023-01-astaria-findings/issues/424) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L251-L265
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L148-L190
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L148


# Vulnerability details

### Description

Some tokens take a transfer fee (e.g. `STA`, `PAXG`), some do not currently charge a fee but may do so in the future (e.g. `USDT`, `USDC`).

Should a fee-on-transfer token be added to the `PublicVault`, the tokens will be locked in the `PublicVault.sol` contract. Depositors will be unable to withdraw their rewards.
In the current implementation, it is assumed that the received amount is the same as the transfer amount. However, due to how fee-on-transfer tokens work, much less will be received than what was transferred.
As a result, later users may not be able to successfully withdraw their shares, as it may revert at https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L148 when `WithdrawProxy` is called due to insufficient balance.

### Proof of Concept

i.e. Fee-on-transfer scenario:
Contract calls transfer from contractA 100 tokens to current contract
Current contract thinks it received 100 tokens
It updates balances to increase +100 tokens
While actually contract received only 90 tokens
That breaks whole math for given token



```solidity
  function deposit(uint256 amount, address receiver)
    public
    override(ERC4626Cloned)
    whenNotPaused
    returns (uint256)
  {
    VIData storage s = _loadVISlot();
    if (s.allowListEnabled) {
      require(s.allowList[receiver]);
    }

    uint256 assets = totalAssets();

    return super.deposit(amount, receiver);
  }
```
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L251-L265

```
  function _redeemFutureEpoch(
    VaultData storage s,
    uint256 shares,
    address receiver,
    address owner,
    uint64 epoch
  ) internal virtual returns (uint256 assets) {
    // check to ensure that the requested epoch is not in the past

    ERC20Data storage es = _loadERC20Slot();

    if (msg.sender != owner) {
      uint256 allowed = es.allowance[owner][msg.sender]; // Saves gas for limited approvals.

      if (allowed != type(uint256).max) {
        es.allowance[owner][msg.sender] = allowed - shares;
      }
    }

    if (epoch < s.currentEpoch) {
      revert InvalidState(InvalidStates.EPOCH_TOO_LOW);
    }
    require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");
    // check for rounding error since we round down in previewRedeem.

    //this will underflow if not enough balance
    es.balanceOf[owner] -= shares;

    // Cannot overflow because the sum of all user
    // balances can't exceed the max uint256 value.
    unchecked {
      es.balanceOf[address(this)] += shares;
    }

    emit Transfer(owner, address(this), shares);
    // Deploy WithdrawProxy if no WithdrawProxy exists for the specified epoch
    _deployWithdrawProxyIfNotDeployed(s, epoch);

    emit Withdraw(msg.sender, receiver, owner, assets, shares);

    // WithdrawProxy shares are minted 1:1 with PublicVault shares
    WithdrawProxy(s.epochData[epoch].withdrawProxy).mint(shares, receiver);
  }
```
https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L148-L190


These functions inherits functions from the `ERC4626-Cloned.sol`
https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626-Cloned.sol

```solidity
  function deposit(uint256 assets, address receiver)
    public
    virtual
    returns (uint256 shares)
  {
    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    require(shares > minDepositAmount(), "VALUE_TOO_SMALL");
    // Need to transfer before minting or ERC777s could reenter.
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    emit Deposit(msg.sender, receiver, assets, shares);

    afterDeposit(assets, shares);
  }

```
https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626-Cloned.sol#L19-L36

### Recommended Mitigation Steps

1. Consider comparing before and after balance to get the actual transferred amount.
2. Alternatively, disallow tokens with fee-on-transfer mechanics to be added as tokens.
