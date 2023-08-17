## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-23

# [Function withdraw() and redeem() in ERC4626RouterBase would revert always because they have unnecessary allowance setting](https://github.com/code-423n4/2023-01-astaria-findings/issues/175) 

# Lines of code

https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626RouterBase.sol#L48
https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626RouterBase.sol#L62


# Vulnerability details

## Impact
Functions withdraw() and redeem()  in ERC4626RouterBase  are used to withdraw user funds from vaults and they call `vault.withdraw()` and `vault.redeem()` and logics in vault transfer user shares and user required to give spending allowance for vault and there is no need for ERC4626RouterBase to set approval for vault and because those approved tokens won't be used and code uses `safeApprove()` so next calls to `withdraw()` and `redeem()` would revert because code would tries to change allowance amount while it's not zero. those functions would revert always and AstariaRouter uses them and user won't be able to use those function and any other protocol integrating with Astaria calling those function would have broken logic. also if UI interact with protocol with router functions then UI would have broken parts too. and functions in router support users to set slippage allowance and without them users have to interact with vault directly and they may lose funds because of the slippage.

## Proof of Concept
This is `withdraw()` and `redeem()` code in ERC4626RouterBase:
```
  function withdraw(
    IERC4626 vault,
    address to,
    uint256 amount,
    uint256 maxSharesOut
  ) public payable virtual override returns (uint256 sharesOut) {

    ERC20(address(vault)).safeApprove(address(vault), amount);
    if ((sharesOut = vault.withdraw(amount, to, msg.sender)) > maxSharesOut) {
      revert MaxSharesError();
    }
  }

  function redeem(
    IERC4626 vault,
    address to,
    uint256 shares,
    uint256 minAmountOut
  ) public payable virtual override returns (uint256 amountOut) {

    ERC20(address(vault)).safeApprove(address(vault), shares);
    if ((amountOut = vault.redeem(shares, to, msg.sender)) < minAmountOut) {
      revert MinAmountError();
    }
  }
```
As you can see the code sets approval for vault to spend routers vault tokens and then call vault function. this is `_redeemFutureEpoch()` code in the vault which handles withdraw and redeem:
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
As you can see this code only check spending allowance that real owner of shares gives to the `msg.sender` and there is no check or updating spending allowance of the router vaulttokens for vault. so those approvals in the `withdraw()` and `redeem()` are unnecessary and they would cause code to revert always because code tries to set approval with `safeApprove()` while the current allowance is not zero.
this issue would cause calls to withdraw() and redeem() function to revert. any other protocol integrating with Astaria using those functions would have broken logic and also users would lose gas if they use those functions. contract AstariaRouter inherits ERC4626RouterBase and uses its `withdraw()` and `redeem()` function so users can't call `AstariaRouter.withdraw()` or `AstariaRouter.redeem()`which supports slippage allowance and they have to call vault's functions directly and they may lose funds because of the slippage.

## Tools Used
VIM

## Recommended Mitigation Steps
remove unnecessary code