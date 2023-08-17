## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-22

# [ERC4626RouterBase.withdraw can only be called once](https://github.com/code-423n4/2023-01-astaria-findings/issues/228) 

# Lines of code

https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626RouterBase.sol#L41-L52


# Vulnerability details

## Impact
ERC4626RouterBase.withdraw will approve an amount of vault tokens to the vault, but the amount represents the number of asset tokens taken out by vault.withdraw, not the required number of vault tokens, and since it normally requires less than 1 vault token to take out 1 asset token, it will prevent ERC4626RouterBase.withdraw from using all approved vault tokens. 
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
and since safeApprove cannot approve a non-zero value to a non-zero value, the second call to ERC4626RouterBase.withdraw will fails in safeApprove.
```solidity
    function safeApprove(
        IERC20 token,
        address spender,
        uint256 value
    ) internal {
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }
```
## Proof of Concept
https://github.com/AstariaXYZ/astaria-gpl/blob/4b49fe993d9b807fe68b3421ee7f2fe91267c9ef/src/ERC4626RouterBase.sol#L41-L52
## Tools Used
None
## Recommended Mitigation Steps
Change to
```diff
  function withdraw(
    IERC4626 vault,
    address to,
    uint256 amount,
-   uint256 maxSharesOut
+  uint256 maxSharesIn
  ) public payable virtual override returns (uint256 sharesOut) {
+   ERC20(address(vault)).safeApprove(address(vault), maxSharesIn);
+   if ((sharesIn = vault.withdraw(amount, to, msg.sender)) > maxSharesIn) {
-   ERC20(address(vault)).safeApprove(address(vault), amount);
-   if ((sharesOut = vault.withdraw(amount, to, msg.sender)) > maxSharesOut) {
      revert MaxSharesError();
    }
+   ERC20(address(vault)).safeApprove(address(vault), 0);

  }
```