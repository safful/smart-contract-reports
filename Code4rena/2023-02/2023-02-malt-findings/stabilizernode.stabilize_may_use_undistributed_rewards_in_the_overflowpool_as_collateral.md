## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- satisfactory
- selected for report
- M-10

# [StabilizerNode.stabilize may use undistributed rewards in the overflowPool as collateral](https://github.com/code-423n4/2023-02-malt-findings/issues/22) 

# Lines of code

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L161-L176


# Vulnerability details

## Impact
In StabilizerNode.stabilize, globalIC.collateralRatio() is used to calculate SwingTraderEntryPrice and ActualPriceTarget, with collateralRatio indicating the ratio of the current global collateral to the malt supply.
```solidity
  function collateralRatio() public view returns (uint256) {
    uint256 decimals = malt.decimals();
    uint256 totalSupply = malt.totalSupply();
    if (totalSupply == 0) {
      return 0;
    }
    return (collateral.total * (10**decimals)) / totalSupply;
  }
```
Global collateral includes the balance of collateral tokens in the overflowPool
```solidity
  function getCollateralizedMalt() public view returns (PoolCollateral memory) {
    uint256 target = maltDataLab.priceTarget(); // 是否选用  getActualPriceTarget()

    uint256 unity = 10**collateralToken.decimals();

    // Convert all balances to be denominated in units of Malt target price
    uint256 overflowBalance = maltDataLab.rewardToMaltDecimals((collateralToken.balanceOf(
      address(overflowPool)
    ) * unity) / target);
    uint256 liquidityExtensionBalance = (collateralToken.balanceOf(
      address(liquidityExtension)
    ) * unity) / target;
    (
      uint256 swingTraderMaltBalance,
      uint256 swingTraderBalance
    ) = swingTraderManager.getTokenBalances();
    swingTraderBalance = (swingTraderBalance * unity) / target;

    return
      PoolCollateral({
        lpPool: address(stakeToken),
        // Note that swingTraderBalance also includes the overflowBalance
        // Therefore the total doesn't need to include overflowBalance explicitly
        total: maltDataLab.rewardToMaltDecimals(
            liquidityExtensionBalance + swingTraderBalance
        ),
```
In StabilizerNode.stabilize, since the undistributed rewards in the overflowPool are not distributed, this can cause the actual collateral radio to be large and thus affect the stabilize process.
A simple example is:
1. impliedCollateralService.syncGlobalCollateral() is called to synchronize the latest data.
2. there are some gap epochs in RewardThrottle and their rewards are not distributed from the overflowPool.
3. When StabilizerNode.stabilize is called, it treats the undistributed rewards in the overflowPool as collateral, thus making globalIC.collateralRatio() large, and the results of maltDataLab. getActualPriceTarget/getSwingTraderEntryPrice will be incorrect, thus making stabilize incorrect.

Since stabilize is a core function of the protocol, stabilizing with the wrong data is likely to cause malt to be depegged, so the vulnerability should be high-risk.

## Proof of Concept
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L161-L176
## Tools Used
None
## Recommended Mitigation Steps
Call RewardThrottle.checkRewardUnderflow at the beginning of StabilizerNode.stabilize to distribute the rewards in the overflowPool, then call impliedCollateralService. syncGlobalCollateral() to synchronize the latest data
```diff
  function stabilize() external nonReentrant onlyEOA onlyActive whenNotPaused {
    // Ensure data consistency
    maltDataLab.trackPool();

    // Finalize auction if possible before potentially starting a new one
    auction.checkAuctionFinalization();

+  RewardThrottle.checkRewardUnderflow();
+  impliedCollateralService.syncGlobalCollateral();

    require(
      block.timestamp >= stabilizeWindowEnd || _stabilityWindowOverride(),
      "Can't call stabilize"
    );
```