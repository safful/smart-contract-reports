## Tags

- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- MR-NEW

# [Reappearance of M-02 in `WstEth.withdraw()`](https://github.com/code-423n4/2023-05-asymmetry-mitigation-findings/issues/70) 

# Reappearance of M-02 in `WstEth.withdraw()`

https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/WstEth.sol#L71-L72

## Description
The changes in [`WstEth.withdraw()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/WstEth.sol#L69C33-L84) has introduced a new issue exactly parallel to the one present in `SfrxEth.withdraw()` which was reported in [M-02: sFrxEth may revert on redeeming non-zero amount](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1049), i.e. `WstEth.withdraw(_amount)` may revert when `_amount > 0`. For why this is an issue please refer to M-02. The mitigation of M-02 was to enable/disable derivatives. See my mitigation review of M-02 for how that issue is not resolved and why I think the mitigation may be insufficient. What is said there equally apply, mutatis mutandis, to this new issue.

## Proof of Concept
`WstEth.withdraw()` now begins
```solidity
uint256 stEthAmount = IWStETH(WST_ETH).unwrap(_amount);
require(stEthAmount > 0, "No stETH to unwrap");
```
We therefore have the same problem as in M-02 if `IWStETH(WST_ETH).unwrap(1) == 0`.
[`WstEth.unwrap()`](https://github.com/lidofinance/lido-dao/blob/df95e563445821988baf9869fde64d86c36be55f/contracts/0.6.12/WstETH.sol#L69-L75) is
```solidity
function unwrap(uint256 _wstETHAmount) external returns (uint256) {
    require(_wstETHAmount > 0, "wstETH: zero amount unwrap not allowed");
    uint256 stETHAmount = stETH.getPooledEthByShares(_wstETHAmount);
    _burn(msg.sender, _wstETHAmount);
    stETH.transfer(msg.sender, stETHAmount);
    return stETHAmount;
}
```
We then ask whether `stETH.getPooledEthByShares(1) == 0`. [StETH.getPooledEthByShares()](https://github.com/lidofinance/lido-dao/blob/df95e563445821988baf9869fde64d86c36be55f/contracts/0.4.24/StETH.sol#L315C14-L324) is:
```solidity
function getPooledEthByShares(uint256 _sharesAmount) public view returns (uint256) {
    uint256 totalShares = _getTotalShares();
    if (totalShares == 0) {
        return 0;
    } else {
        return _sharesAmount
            .mul(_getTotalPooledEther())
            .div(totalShares);
    }
}
```
So just like in M-02, if `_getTotalPooledEther() < totalShares` then `IWStETH(WST_ETH).unwrap(1) == 0` and `WstEth.withdraw(1)` reverts.

## Recommended Mitigation Steps
Replace `require(stEthAmount > 0, "No stETH to unwrap");` with `if (stEthAmount > 0) return;`