## Tags

- bug
- 1 (Low Risk)
- sponsor acknowledged
- sponsor confirmed

# [No ERC20 safeApprove called & not success check](https://github.com/code-423n4/2021-11-unlock-findings/issues/161) 

# Handle

cmichel


# Vulnerability details

Some tokens (like USDT) don't correctly implement the EIP20 standard and their `approve` function returns `void` instead of a success boolean. Calling these functions with the correct EIP20 function signatures will always revert.

For the tokens that return a success value, the contract does not check it.

Non-safe transfers are used in:
- `MixinLockCore.approveBeneficiary`: `IERC20Upgradeable(tokenAddress).approve(_spender, _amount)`


## Impact
Tokens that return `false` on a failed `approve` or that don't correctly implement the latest EIP20 spec, like USDT, will be unusable in the protocol as they revert the transaction because of the missing return value.

## Recommended Mitigation Steps
We recommend using OpenZeppelin’s `SafeERC20` versions with the `safeApprove` function that handle the return value check as well as non-standard-compliant tokens.

