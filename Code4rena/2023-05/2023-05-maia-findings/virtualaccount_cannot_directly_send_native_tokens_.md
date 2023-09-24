## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-32

# [VirtualAccount cannot directly send native tokens ](https://github.com/code-423n4/2023-05-maia-findings/issues/307) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/VirtualAccount.sol#L41-L53


# Vulnerability details

## Impact
Certain functions require native tokens to be sent. These functions will revert.

## Proof of Concept
According to the Sponsor, VirtualAccounts can "call any of the dApps present in the Root Chain (Arbitrum) e.g. Maia, Hermes, Ulysses AMM,Uniswap." However, this is not the case as `call()` is not `payable` and thus cannot send native tokens to other contracts. This is problematic because certain [functions](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BaseBranchRouter.sol#L58-L62) require native token transfers and will fail.

## Tools Used
Manual

## Recommended Mitigation Steps
Consider creating a single `call()` function that has a `payable` modifier and `{value: msg.value}`. Be aware that since `calls[i].target.call()` is in a loop, it is not advisable to add payable to the existing `call()`. This is because  `msg.value` may be used multiple times, and is [unsafe](https://github.com/Uniswap/v3-periphery/issues/52).








## Assessed type

Payable