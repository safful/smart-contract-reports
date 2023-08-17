## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-03

# [Rewards will be locked in LQTYStaking Contract](https://github.com/code-423n4/2023-02-ethos-findings/issues/285) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/LQTY/LQTYStaking.sol#L181-L183
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/RedemptionHelper.sol#L191-L197
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L296-L300


# Vulnerability details

# Vulnerability details
The state variable `F_Collateral` in the LQTYStaking contract is used to keep track of rewards for each of the collateral types used in the protocol. Every time the LQTYStaking contract is sent collateral assets for rewards by the ActivePool or the RedemptionHelper, `LQTYStaking.increaseF_Collateral` is called to record the rewards that are to be distributed to stakers. 

However, if the state variable `totalLQTYStaked` is large enough in the LQTYStaking contract, zero rewards will be distributed to stakers even though LQTYStaking received assets. This issue is exarcebated when using WBTC as collateral due to its low number of decimals. 

For example, given the following:
1. `totalLQTYStaked` = 1e25; LQTY/OATH token has 18 decimals; this means that a total of 10million LQTY has been staked
2. A redemption rate of 0.5% was applied on a redemption of 10e8 WBTC. This leads to a redemption fee of 5e6 WBTC that is sent to the LQTYStaking contract. This happens in [this code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/RedemptionHelper.sol#L190-L197). 
3. Given the above, RedemptionHelper calls `LQTYStaking.increaseF_Collateral(WBTCaddress, 5e6)`

The issue is in this line in `increaseF_Collateral`:
```solidity
if (totalLQTYStaked > 0) {collFeePerLQTYStaked = _collFee.mul(DECIMAL_PRECISION).div(totalLQTYStaked);}
```

`_collFee` = 5e6; `DECIMAL_PRECISION` = 1e18; `totalLQTYStaked` = 1e25

If we substitute the variables with the actual values and represent the code in math, it looks like:
```
(5e6 * 1e18) / 1e25 = 5e24 / 1e25 = 0.5
```

Since the result of that math is a value less than 1 and in Solidity/EVM we only deal with integers and division rounds down, we get 0 as a result. That means the below code will only add `0` to `F_Collateral`:
```solidity
F_Collateral[_collateral] = F_Collateral[_collateral].add(collFeePerLQTYStaked);
```

So even though LQTYStaking received 5e6 WBTC in redemption fee, that fee will never be distributed to stakers and will remain forever locked in the LQTYStaking contract. The minimum amount of redemption fee that is needed for the reward to be recognized and distributed to stakers is 1e7 WBTC. That means at least 0.1 BTC in collateral fee is needed for the rewards to be distributed when there is 1Million total LQTY staked.


## Impact

This leads to loss of significant rewards for stakers. These collateral assets that are not distributed as rewards will remain forever locked in LQTYStaking.

If 1e25 LQTY is staked in LQTYStaking (10M LQTY), at least 1e7 (0.1) WBTC in redemption fee must be sent by the RedemptionHelper for that WBTC to be sent as rewards to the stakers. That means only redemptions of 20e8 (20) WBTC and more will lead to redemption fees high enough to be distributed as rewards to stakers. Redemption of 20e8 WBTC will rarely happen, so it's likely that majority of rewards will be forever locked since most redemptions will be less than that.

Given the above, if only 3% of redemptions have amounts of 20e8 WBTC or greater, then 97% of redemptions will have their fees forever locked in the contract. The greater the amount of LQTY Staked, the higher the amount needed for the fees to be recorded. 

## Proof of Concept

First, comment out [this line](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/LQTY/LQTYStaking.sol#L178) in `increaseF_Collateral` to disable the access control. This allows us to write a more concise POC. It is fine since the issue has nothing to do with access control.

Add the following test case to the `Ethos-Core/test/LQTYStakingFeeRewardsTest.js` file after the `beforeEach` clause:
```js
  it('does not increase F collateral even with large amount of collateral fee', async () => {
    await stakingToken.mint(A, dec(10_000_000, 18))
    await stakingToken.approve(lqtyStaking.address, dec(10_000_000, 18), {from: A})
    await lqtyStaking.stake(dec(10_000_000, 18), {from: A})

    const wbtc = collaterals[1].address
    const oldWBTC_FCollateral = await lqtyStaking.F_Collateral(wbtc)

    // .09 WBTC in redemption/collateral fee will not be distributed as reward to stakers
    await lqtyStaking.increaseF_Collateral(wbtc, dec(9, 6))
    assert.isTrue(oldWBTC_FCollateral.eq(await lqtyStaking.F_Collateral(wbtc)))
    
    // at least 0.1 WBTC in redemption/collateral fee is needed for it to be distributed as reward to stakers
    await lqtyStaking.increaseF_Collateral(wbtc, dec(1, 7))
    assert.isTrue(oldWBTC_FCollateral.lt(await lqtyStaking.F_Collateral(wbtc)))
  })
```

The test can then be run with the following command:
```
$ npx hardhat test --grep "does not increase F collateral even with large amount of collateral fee"
```

## Tools Used
Manual review

## Mitigation
One way to address this issue is to use the same error-recording logic found in the `_computeLQTYPerUnitStaked` logic that looks like:

```solidity
        uint LQTYNumerator = _LQTYIssuance.mul(DECIMAL_PRECISION).add(lastLQTYError);

        uint LQTYPerUnitStaked = LQTYNumerator.div(_totalLUSDDeposits);
        lastLQTYError = LQTYNumerator.sub(LQTYPerUnitStaked.mul(_totalLUSDDeposits));
```

The `lastLQTYError` state variable stores the LQTY issuance that was not distributed since they were just rounded off. The same approach can be used in `increaseF_Collateral`.