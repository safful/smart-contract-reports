## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- low quality report
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [sFrxEth may revert on redeeming non-zero amount](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1049) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L61-L65
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L118


# Vulnerability details




## Impact
Unstaking is blocked.

## Proof of Concept
When unstaking the `withdraw` of each derivative is called. `SfrxEth.withdraw` calls [`IsFrxEth(SFRX_ETH_ADDRESS).redeem(_amount, address(this), address(this));`](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/SfrxEth.sol#L61-L65). This function may revert if `_amount` is low due to the following line in `redeem` (where `_amount` is `shares`):
[`require((assets = previewRedeem(shares)) != 0, "ZERO_ASSETS");`](https://etherscan.io/token/0xac3e018457b222d93114458476f3e3416abbe38f#code#L708)
`previewRedeem(uint256 shares)` [returns `convertToAssets(shares)`](https://etherscan.io/token/0xac3e018457b222d93114458476f3e3416abbe38f#code#L753) which is the shares scaled by the division of total assets by total supply:
[`shares.mulDivDown(totalAssets(), supply)`](https://etherscan.io/token/0xac3e018457b222d93114458476f3e3416abbe38f#code#L733).
So if `_amount == 1` and total assets in sFrxEth is less than its total supply, then `previewRedeem(shares) == 0` and `redeem` will revert. This revert in `SfrxEth.withdraw` causes a revert in `SafEth.unstake` at [L118](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L118), which means that funds cannot be unstaked.

`_amount` may be as low as 1 when the weight for this derivative has been set to 0 and funds have adjusted over time through staking and unstaking until only 1 remains in the SfrxEth derivative. Instead of just being depleted it may thus block unstaking.

## Recommended Mitigation Steps
In `SfrxEth.withdraw` check if `IsFrxEth(SFRX_ETH_ADDRESS).previewRedeem(_amount) == 0` and simply return if that's the case. 