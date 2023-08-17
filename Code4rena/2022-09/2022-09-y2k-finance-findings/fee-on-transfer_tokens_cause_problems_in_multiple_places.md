## Tags

- bug
- 2 (Med Risk)
- sponsor acknowledged
- selected for report

# [Fee-on-Transfer tokens cause problems in multiple places](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/221) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/bca5080635370424a9fe21fe1aded98345d1f723/src/SemiFungibleVault.sol#L94
https://github.com/code-423n4/2022-09-y2k-finance/blob/bca5080635370424a9fe21fe1aded98345d1f723/src/Controller.sol#L168
https://github.com/code-423n4/2022-09-y2k-finance/blob/bca5080635370424a9fe21fe1aded98345d1f723/src/Controller.sol#L225


# Vulnerability details

## Impact & Proof Of Concept
Certain tokens (e.g., STA or PAXG) charge a fee for transfers and others (e.g., USDT or USDC) may start doing so in the future. This is not correctly handled in multiple places and would lead to a loss of funds:
1.) `SemiFungibleVault.deposit`: Here, less tokens are transferred to the vault than the amount of shares that is minted to the user. This is an accounting mistake that will ultimately lead to the situation where the last user(s) cannot withdraw anymore, because there are no more assets left.
2.) `Controller.triggerDepeg` & `Controller.triggerEndEpoch`: `sendTokens` tries to send the whole asset balance to the other contract, which will fail when less tokens are available at this point (because the previous accounting was done without incorporating fees). This will mean that the end can never be triggered and all assets are lost.

## Recommended Mitigation Steps
When fee-on-transfer tokens should be supported, you need to check the actual balance differences. If they are not supported, this should be clearly documented.