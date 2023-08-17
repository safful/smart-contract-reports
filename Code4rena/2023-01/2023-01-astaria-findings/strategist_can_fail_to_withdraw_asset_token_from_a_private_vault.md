## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-04

# [Strategist can fail to withdraw asset token from a private vault](https://github.com/code-423n4/2023-01-astaria-findings/issues/489) 

# Lines of code

https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626RouterBase.sol#L41-L52
https://github.com/code-423n4/2023-01-astaria/blob/main/src/Vault.sol#L70-L73


# Vulnerability details

## Impact
Calling the `AstariaRouter.withdraw` function calls the following `ERC4626RouterBase.withdraw` function; however, calling `ERC4626RouterBase.withdraw` function for a private vault reverts because the `Vault` contract does not have an `approve` function. Directly calling the `Vault.withdraw` function for a private vault can also revert since the `Vault` contract does not have a way to set the allowance for itself to transfer the asset token, which can cause many ERC20 tokens' `transferFrom` function calls to revert when deducting the transfer amount from the allowance. Hence, after depositing some of the asset token in a private vault, the strategist can fail to withdraw this asset token from this private vault and lose this deposit.

https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626RouterBase.sol#L41-L52
```solidity
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
```

https://github.com/code-423n4/2023-01-astaria/blob/main/src/Vault.sol#L70-L73
```solidity
  function withdraw(uint256 amount) external {
    require(msg.sender == owner());
    ERC20(asset()).safeTransferFrom(address(this), msg.sender, amount);
  }
```

## Proof of Concept
Please add the following test in `src\test\AstariaTest.t.sol`. This test will pass to demonstrate the described scenario.

```solidity
  function testPrivateVaultStrategistIsUnableToWithdraw() public {
    uint256 amountToLend = 50 ether;
    vm.deal(strategistOne, amountToLend);

    address privateVault = _createPrivateVault({
      strategist: strategistOne,
      delegate: strategistTwo
    });

    vm.startPrank(strategistOne);

    WETH9.deposit{value: amountToLend}();
    WETH9.approve(privateVault, amountToLend);

    // strategistOne deposits 50 ether WETH to privateVault
    Vault(privateVault).deposit(amountToLend, strategistOne);

    // calling router's withdraw function for withdrawing assets from privateVault reverts
    vm.expectRevert(bytes("APPROVE_FAILED"));
    ASTARIA_ROUTER.withdraw(
      IERC4626(privateVault),
      strategistOne,
      amountToLend,
      type(uint256).max
    );

    // directly withdrawing various asset amounts from privateVault also fails
    vm.expectRevert(bytes("TRANSFER_FROM_FAILED"));
    Vault(privateVault).withdraw(amountToLend);

    vm.expectRevert(bytes("TRANSFER_FROM_FAILED"));
    Vault(privateVault).withdraw(1);

    vm.stopPrank();
  }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
https://github.com/code-423n4/2023-01-astaria/blob/main/src/Vault.sol#L72 can be updated to the following code.

```solidity
    ERC20(asset()).safeTransfer(msg.sender, amount);
```