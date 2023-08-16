## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [ERC20 tokens with no return value will fail to transfer](https://github.com/code-423n4/2022-03-joyn-findings/issues/83) 

# Lines of code

https://github.com/code-423n4/2022-03-joyn/blob/main/royalty-vault/contracts/RoyaltyVault.sol#L43-L46
https://github.com/code-423n4/2022-03-joyn/blob/main/royalty-vault/contracts/RoyaltyVault.sol#L51-L57


# Vulnerability details

 Although the ERC20 standard suggests that a transfer should return true on success, many tokens are non-compliant in this regard (including high profile, like USDT) . In that case, the .transfer() call here will revert even if the transfer is successful, because solidity will check that the RETURNDATASIZE matches the ERC20 interface.

Recommendation: Consider using OpenZeppelin’s SafeERC20

