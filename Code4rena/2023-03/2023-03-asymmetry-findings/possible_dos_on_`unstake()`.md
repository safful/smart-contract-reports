## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [Possible DoS on `unstake()`](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/685) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L107-L114


# Vulnerability details

## Impact
RocketPool rETH tokens have a [deposit delay](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L157-L172) that prevents any user who has recently deposited to transfer or burn tokens. In the past this delay was set to 5760 blocks mined (aprox. 19h, considering one block per 12s). This delay can prevent asymmetry protocol users from unstaking if another user staked recently.

While it's not currently possible due to RocketPool's configuration, any future changes made to this delay by the admins could potentially lead to a denial-of-service attack on the `ùnstake()` mechanism. This is a major functionality of the asymmetry protocol, and therefore, it should be classified as a high severity issue.

## Proof of Concept
Currently, the delay is set to zero, but if RocketPool admins decide to change this value in the future, it could cause issues. Specifically, protocol users staking actions could prevent other users from unstaking for a few hours. Given that many users call the stake function throughout the day, the delay would constantly reset, making the unstaking mechanism unusable. It's important to note that this only occurs when `stake()` is used through the `rocketDepositPool` route. If rETH is obtained from the Uniswap pool, the delay is not affected.

A malicious actor can also exploit this to be able to block all `unstake` calls. Consider the following scenario where the delay was raised again to 5760 blocks. Bob (malicious actor) call `stakes()` with the minimum amount, consequently triggering deposit to RocketPool and resetting the deposit delay. Alice tries to `unstake` her funds, but during rETH burn, it fails due to the delay check, reverting the `unstake` call.

If Bob manages to repeatedly `stakes()` the minimum amount every 19h (or any other interval less then the deposit delay), all future calls to `unstake` will revert.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider modifying Reth derivative to obtain rETH only through the UniswapV3 pool, on average users will get less rETH due to the slippage, but will avoid any future issues with the deposit delay mechanism.