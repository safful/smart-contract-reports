## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-08

# [Pausing `WardenPledge` contract, which takes effect immediately, by its owner can unexpectedly block pledge creator from calling `closePledge` or `retrievePledgeRewards` function](https://github.com/code-423n4/2022-10-paladin-findings/issues/269) 

# Lines of code

https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L636-L638
https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L488-L515
https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L456-L480


# Vulnerability details

## Impact
The owner of the `WardenPledge` contract is able to call the `pause` function to pause this contract. When the `WardenPledge` contract is paused, calling the `closePledge` or `retrievePledgeRewards` function that uses the `whenNotPaused` modifier reverts, and the pledge creator is not able to get back any of the reward token amount, which was deposited by the creator previously. Because calling the `pause` function takes effect immediately, it can be unexpected to the creator for suddenly not being able to call the `closePledge` or `retrievePledgeRewards` function. For instance, when an emergency occurs that requires an increase of cash flow, the creator wants to close the pledge early so she or he can use the remaining deposited reward token amount. However, just before the creator's `closePledge` transaction is executed, the `pause` transaction has been sent by the owner of the `WardenPledge` contract for some reason and executed. Without knowing in advance that the `WardenPledge` contract would be paused, the creator anticipates to receive the remaining deposited reward token amount but this is not the case since calling the `closePledge` function reverts. Because the creator unexpectedly fails to receive such amount and might fail to deal with the emergency, disputes with the protocol can occur, and the user experience becomes degraded.

https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L636-L638
```solidity
    function pause() external onlyOwner {
        _pause();
    }
```

https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L488-L515
```solidity
    function closePledge(uint256 pledgeId, address receiver) external whenNotPaused nonReentrant {
        ...

        // Get the current remaining amount of rewards not distributed for the Pledge
        uint256 remainingAmount = pledgeAvailableRewardAmounts[pledgeId];

        if(remainingAmount > 0) {
            // Transfer the non used rewards and reset storage
            pledgeAvailableRewardAmounts[pledgeId] = 0;

            IERC20(pledgeParams.rewardToken).safeTransfer(receiver, remainingAmount);

            ...

        }

        ...
    }
```

https://github.com/code-423n4/2022-10-paladin/blob/main/contracts/WardenPledge.sol#L456-L480
```solidity
    function retrievePledgeRewards(uint256 pledgeId, address receiver) external whenNotPaused nonReentrant {
        ...

        // Get the current remaining amount of rewards not distributed for the Pledge
        uint256 remainingAmount = pledgeAvailableRewardAmounts[pledgeId];

        ...

        if(remainingAmount > 0) {
            // Transfer the non used rewards and reset storage
            pledgeAvailableRewardAmounts[pledgeId] = 0;

            IERC20(pledgeParams.rewardToken).safeTransfer(receiver, remainingAmount);

            ...

        }
    }
```

## Proof of Concept
Please append the following test in the `pause & unpause` `describe` block in `test\wardenPledge.test.ts`. This test will pass to demonstrate the described scenario.

```typescript
        it.only('Pausing WardenPledge contract, which takes effect immediately, by its owner can unexpectedly block pledge creator from calling closePledge function', async () => {
            // before calling the createPledge function, the wardenPledge contract owns no rewardToken1
            const rewardToken1BalanceWardenPledgeBefore = await rewardToken1.balanceOf(wardenPledge.address)
            expect(rewardToken1BalanceWardenPledgeBefore).to.be.eq(0)

            const rewardToken1BalanceCreatorBefore = await rewardToken1.balanceOf(creator.address)

            // creator calls the createPledge function
            await wardenPledge.connect(creator).createPledge(
                receiver.address,
                rewardToken1.address,
                target_votes,
                reward_per_vote,
                end_timestamp,
                max_total_reward_amount,
                max_fee_amount
            )

            // after one week, admin, who is the owner of the wardenPledge contract, calls the pause function, which takes effect immediately
            await advanceTime(WEEK.toNumber())
            await wardenPledge.connect(admin).pause()

            // Since an emergency that requires an increase of cash flow occurs, creator decides to close the pledge for getting back the deposited rewardToken1 amount.
            // Without knowing in advance that the wardenPledge contract would be paused,
            //   creator calls the closePledge function and anticipates to receive the deposited rewardToken1 amount.
            // Unfortunately, admin's pause transaction has been executed just before creator's closePledge transaction is executed, which causes creator's closePledge transaction to revert.
            await expect(
                wardenPledge.connect(creator).closePledge(pledge_id, creator.address)
            ).to.be.revertedWith("Pausable: paused")

            // after creator's closePledge transaction reverts, creator does not receive the deposited rewardToken1 amount, which is unexpected to her or him
            const rewardToken1BalanceCreatorAfter = await rewardToken1.balanceOf(creator.address)
            expect(rewardToken1BalanceCreatorAfter).to.be.lt(rewardToken1BalanceCreatorBefore)

            // meanwhile, the wardenPledge contract still holds the creator's deposited rewardToken1 amount
            const rewardToken1BalanceWardenPledgeAfter = await rewardToken1.balanceOf(wardenPledge.address)
            expect(rewardToken1BalanceWardenPledgeAfter).to.be.gt(0)
        });
```

## Tools Used
VSCode

## Recommended Mitigation Steps
The `pause` function can be updated to be time-delayed so the pledge creator can have more time to react. One way would be making this function only callable by a timelock governance contract.