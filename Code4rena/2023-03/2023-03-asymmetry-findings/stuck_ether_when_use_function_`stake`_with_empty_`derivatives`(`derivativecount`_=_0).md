## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-10

# [Stuck ether when use function `stake` with empty `derivatives`(`derivativeCount` = 0)](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/363) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L84-L96


# Vulnerability details

### Impact

After `initialize` the contract `SafEth`, if someone call `stake` before `addDerivative`, the function `stake` skip the two for cycles because the `derivativeCount` is equal to `0` and don't `deposit` in the `derivative` contract also mint `0` tokens to the sender. Finally the amount of `msg.value` will stuck in the contract

### Proof of Concept

```typescript
/* eslint-disable new-cap */
import { network, upgrades, ethers } from "hardhat";
import { expect } from "chai";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { SafEth } from "../typechain-types";

describe("stake tests", function () {
  let adminAccount: SignerWithAddress;
  let safEthProxy: SafEth;
  const depositAmount = ethers.utils.parseEther("200");

  before(async () => {
    const latestBlock = await ethers.provider.getBlock("latest");

    await network.provider.request({
      method: "hardhat_reset",
      params: [{forking: {
        jsonRpcUrl: process.env.MAINNET_URL,
        blockNumber: latestBlock.number,
      }}],
    });

    const accounts = await ethers.getSigners();
    adminAccount = accounts[0];

    safEthProxy = await upgrades.deployProxy(
      await ethers.getContractFactory("SafEth"),
      [
        "Asymmetry Finance ETH",
        "safETH",
      ]
    ) as SafEth;
    await safEthProxy.deployed();
  });

  it("PoC: don't have derivatives", async function () {
    // Check: don't have derivatives
    expect(await safEthProxy.derivativeCount()).eq(0);

    // This transaction should revert
    await safEthProxy.stake({ value: depositAmount });

    const ethBal = await ethers.provider.getBalance(safEthProxy.address);
    const stakerBal = await safEthProxy.balanceOf(adminAccount.address);
    // This log 200 ether, but should be 0
    console.log("safEthProxy Balance:", ethBal.toString());
    // The staker has 0 tokens
    console.log("staker Balance:", stakerBal.toString());
  });
});
```

### Tools Used

Review

### Recommended Mitigation Steps

When stake the `derivativeCount` should be greater than `0`:

```solidity
@@ -64,6 +64,7 @@ contract SafEth is
         require(pauseStaking == false, "staking is paused");
         require(msg.value >= minAmount, "amount too low");
         require(msg.value <= maxAmount, "amount too high");
+        require(derivativeCount > 0, "derivativeCount is zero");

         uint256 underlyingValue = 0;
```