## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- edited-by-warden
- H-05

# [_distributeProfit will use the stale globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio(), which will result in incorrect profit distribution](https://github.com/code-423n4/2023-02-malt-findings/issues/10) 

# Lines of code

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/ProfitDistributor.sol#L164-L184


# Vulnerability details

## Impact
The _distributeProfit() (called by handleProfit()) will use globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio() when distributing profits, and the latest globalIC.swingTraderCollateralDeficit()/swingTraderCollateralRatio() needs to be used to ensure that profits are distributed correctly. 
```solidity
    uint256 globalSwingTraderDeficit = (maltDataLab.maltToRewardDecimals(
      globalIC.swingTraderCollateralDeficit()
    ) * maltDataLab.priceTarget()) / (10**collateralToken.decimals());

    // this is already in collateralToken.decimals()
    uint256 lpCut;
    uint256 swingTraderCut;

    if (globalSwingTraderDeficit == 0) {
      lpCut = distributeCut;
    } else {
      uint256 runwayDeficit = rewardThrottle.runwayDeficit();

      if (runwayDeficit == 0) {
        swingTraderCut = distributeCut;
      } else {
        uint256 totalDeficit = runwayDeficit + globalSwingTraderDeficit;
```
However, the two calls to handleProfit in the contract do not call syncGlobalCollateral to synchronize the data in globalIC.
syncGlobalCollateral will use the data in getCollateralizedMalt(), including the collateralToken balance in overflowPool/swingTraderManager/liquidityExtension and the malt balance in swingTraderManager.
```
  function syncGlobalCollateral() public onlyActive {
    globalIC.sync(getCollateralizedMalt());
  }

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
```
1. before handleProfit is called by StabilizerNode.stabilize.
```solidity
    profitDistributor.handleProfit(rewards);
```
a. checkAuctionFinalization is called to liquidityExtension.allocateBurnBudget, which transfers the collateralToken from liquidityExtension to swingTrader. The increase of collateralToken in swingTrader will make the data in globalIC stale.
```solidity
function allocateBurnBudget(uint256 amount)
    external
    onlyRoleMalt(AUCTION_ROLE, "Must have auction privs")
    onlyActive
    returns (uint256 purchased)
  {
    // Send the burnable amount to the swing trader so it can be used to burn more malt if required
    require(
      collateralToken.balanceOf(address(this)) >= amount,
      "LE: Insufficient balance"
    );
    collateralToken.safeTransfer(address(swingTrader), amount);

    emit AllocateBurnBudget(amount);
  }
```
b. swingTraderManager.sellMalt will exchange malt for collateralToken, and the increase of collateralToken in swingTrader will also make the data in globalIC stale.
```solidity
    uint256 swingAmount = swingTraderManager.sellMalt(tradeSize);
```

2. before SwingTrader.sellMalt is called to handleProfit.
```solidity
  function _handleProfitDistribution(uint256 profit) internal virtual {
    if (profit != 0) {
      collateralToken.safeTransfer(address(profitDistributor), profit);
      profitDistributor.handleProfit(profit);
    }
  }
```
a. dexHandler.sellMalt will exchange malt for collateralToken, and the increase of collateralToken in swingTrader will also make the data in globalIC stale.
```solidity
    malt.safeTransfer(address(dexHandler), maxAmount);
    uint256 rewards = dexHandler.sellMalt(maxAmount, 10000);

```
One obvious effect is that as the collateralToken in swingTrader increases, collateral.swingTrade will be smaller than it actually is, and the result of globalIC.swingTraderCollateralDeficit() will be larger than it should be.
```solidity
  function swingTraderCollateralDeficit() public view returns (uint256) {
    // Note that collateral.swingTrader is already denominated in malt.decimals()
    uint256 maltSupply = malt.totalSupply();
    uint256 collateral = collateral.swingTrader; // gas

    if (collateral >= maltSupply) {
      return 0;
    }

    return maltSupply - collateral;
  }
```
thus making lpCut larger
```solidity
    uint256 globalSwingTraderDeficit = (maltDataLab.maltToRewardDecimals(
      globalIC.swingTraderCollateralDeficit()
    ) * maltDataLab.priceTarget()) / (10**collateralToken.decimals());

    // this is already in collateralToken.decimals()
    uint256 lpCut;
    uint256 swingTraderCut;

    if (globalSwingTraderDeficit == 0) {
      lpCut = distributeCut;
    } else {
      uint256 runwayDeficit = rewardThrottle.runwayDeficit();

      if (runwayDeficit == 0) {
        swingTraderCut = distributeCut;
      } else {
        uint256 totalDeficit = runwayDeficit + globalSwingTraderDeficit;

        uint256 globalSwingTraderRatio = maltDataLab.maltToRewardDecimals(
          globalIC.swingTraderCollateralRatio()
        );

        // Already in collateralToken.decimals
        uint256 poolSwingTraderRatio = impliedCollateralService
          .swingTraderCollateralRatio();

        if (poolSwingTraderRatio < globalSwingTraderRatio) {
          swingTraderCut = (distributeCut * swingTraderPreferenceBps) / 10000;
          lpCut = distributeCut - swingTraderCut;
        } else {
          lpCut =
            (((distributeCut * runwayDeficit) / totalDeficit) *
              (10000 - lpThrottleBps)) /
            10000;
```
## Proof of Concept
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/ProfitDistributor.sol#L164-L184
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L423-L424
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTrader.sol#L176-L181
## Tools Used
None
## Recommended Mitigation Steps
 Call syncGlobalCollateral to synchronize the data in globalIC before calling handleProfit.