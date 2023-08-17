## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- upgraded by judge
- H-01

# [griefing / blocking / delaying users to withdraw](https://github.com/code-423n4/2022-12-prepo-findings/issues/116) 

# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L66-L72


# Vulnerability details

## griefing / blocking / delaying users to withdraw
To withdraw, a user needs to convert his collateral for the base token. This is done in the **withdraw** function in Collateral.  
The WithdrawHook has some security mechanics that can be activated like a global max withdraw in a specific timeframe, also for users to have a withdraw limit for them in a specific timeframe. It also collects the fees.

The check for the user withdraw is wrongly implemented and can lead to an unepexted delay for a user with a position **> userWithdrawLimitPerPeriod**. To withdraw all his funds he needs to be the first in every first new epoch (**lastUserPeriodReset** + **userPeriodLength**) to get his amount out. If he is not the first transaction in the new epoch, he needs to wait for a complete new epoch and depending on the timeframe from **lastUserPeriodReset** + **userPeriodLength** this can get a long delay to get his funds out.

The documentation says, that after every epoch all the user withdraws will be reset and they can withdraw the next set.

```solidity
File: apps/smart-contracts/core/contracts/interfaces/IWithdrawHook.sol
63:   /**
64:    * @notice Sets the length in seconds for which user withdraw limits will
65:    * be evaluated against. Every time `userPeriodLength` seconds passes, the
66:    * amount withdrawn for all users will be reset to 0. This amount is only
```

But the implementation only resets the amount for the first user that interacts with the contract in the new epoch and leaves all other users with their old limit. This can lead to a delay for every user that is on his limit from a previous epoch until they manage to be the first to interact with the contract in the new epoch.

## Proof of Concept
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L66-L72

The following test shows how a user is locked out to withdraw if he's at his limit from a previous epoch and another withdraw is done before him.

apps/smart-contracts/core/test/WithdrawHook.test.ts  
```node
  describe('user withdraw is delayd', () => {
    beforeEach(async () => {
      await withdrawHook.setCollateral(collateral.address)
      await withdrawHook.connect(deployer).setWithdrawalsAllowed(true)
      await withdrawHook.connect(deployer).setGlobalPeriodLength(0)
      await withdrawHook.connect(deployer).setUserPeriodLength(TEST_USER_PERIOD_LENGTH)
      await withdrawHook.connect(deployer).setGlobalWithdrawLimitPerPeriod(0)
      await withdrawHook.connect(deployer).setUserWithdrawLimitPerPeriod(TEST_USER_WITHDRAW_LIMIT)
      await withdrawHook.connect(deployer).setDepositRecord(depositRecord.address)
      await withdrawHook.connect(deployer).setTreasury(treasury.address)
      await withdrawHook.connect(deployer).setTokenSender(tokenSender.address)
      await testToken.connect(deployer).mint(collateral.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken.connect(deployer).mint(user.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken.connect(deployer).mint(user2.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken
        .connect(collateralSigner)
        .approve(withdrawHook.address, ethers.constants.MaxUint256)
      tokenSender.send.returns()
    })

    it('reverts if user withdraw limit exceeded for period', async () => {
      
      // first withdraw with the limit amount for a user
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      expect(await withdrawHook.getAmountWithdrawnThisPeriod(user.address)).to.eq(TEST_USER_WITHDRAW_LIMIT)
      
      // we move to a new epoch in the future
      const previousResetTimestamp = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp + TEST_USER_PERIOD_LENGTH + 1
      )
      
      // now another user is the first one to withdraw in this new epoch      
      await withdrawHook.connect(collateralSigner).hook(user2.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      expect(await withdrawHook.getAmountWithdrawnThisPeriod(user2.address)).to.eq(TEST_USER_WITHDRAW_LIMIT)
      
      // this will revert, because userToAmountWithdrawnThisPeriod[_sender] is not reset
      // but it should not revert as it's a new epoch and the user didn't withdraw yet
      await expect(
        withdrawHook.connect(collateralSigner).hook(user.address, 1, 1)
      ).to.revertedWith('user withdraw limit exceeded')
      
    })
  })
```
To get the test running you need to add **let user2: SignerWithAddress** and the user2 in **await ethers.getSigners()**

## Recommended Mitigation Steps
The check how the user periods are handled need to be changed. One possible way is to change the lastUserPeriodReset to a mapping like
**mapping(address => uint256) private lastUserPeriodReset** to track the time for every user seperatly.

With a mapping you can change the condition to
```solidity
File: apps/smart-contracts/core/contracts/WithdrawHook.sol
18:   mapping(address => uint256) lastUserPeriodReset;

File: apps/smart-contracts/core/contracts/WithdrawHook.sol
66:     if (lastUserPeriodReset[_sender] + userPeriodLength < block.timestamp) {
67:       lastUserPeriodReset[_sender] = block.timestamp;
68:       userToAmountWithdrawnThisPeriod[_sender] = _amountBeforeFee;
69:     } else {
70:       require(userToAmountWithdrawnThisPeriod[_sender] + _amountBeforeFee <= userWithdrawLimitPerPeriod, "user withdraw limit exceeded");
71:       userToAmountWithdrawnThisPeriod[_sender] += _amountBeforeFee;
72:     }
```

With this change, we can change the test to how we would normaly expect the contract to work and see that it is correct.

```node
    it('withdraw limit is checked for every use seperatly', async () => {
      
      // first withdraw with the limit amount for a user
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // we move to a new epoch in the future
      const previousResetTimestamp = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp + TEST_USER_PERIOD_LENGTH + 1
      )
      
      // now another user is the first one to withdraw in this new epoch      
      await withdrawHook.connect(collateralSigner).hook(user2.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // the first user also can withdraw his limit in this epoch
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // we move the time, but stay in the same epoch
      const previousResetTimestamp2 = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp2 + TEST_USER_PERIOD_LENGTH - 1
      )

      // this now will fail as we're in the same epoch
      await expect(
        withdrawHook.connect(collateralSigner).hook(user.address, 1, 1)
      ).to.revertedWith('user withdraw limit exceeded')
      
    })
```