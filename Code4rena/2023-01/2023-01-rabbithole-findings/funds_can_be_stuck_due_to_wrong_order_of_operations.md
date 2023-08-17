## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-07

# [Funds can be stuck due to wrong order of operations](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/122) 

# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102-L104
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L81-L87


# Vulnerability details

## Impact
The contract `ERC20Quest.sol` has two functions of interest here. The first is `withdrawFee()`, which is responsible for transferring out the fee amount from the contract once endTime has been passed, and the second is `withdrawRemainingTokens()` which recovers the remaining tokens in the contract which haven't been claimed yet.

Function `withdrawRemainingTokens()`:
```solidity
function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
    }
```

As evident from this excerpt, calling this recovery function subtracts the tokens which are already assigned to someone who completed the quest, and the fee, and returns the rest. However, there is no check for whether the fee has already been paid or not. The owner is expected to first call `withdrawRemainingTokens()`, and then call `withdrawFee()`.

However, if the owner calls `withdrawFee()` before calling the function `withdrawRemainingTokens()`, the fee will be paid out by the first call, but the same fee amount will still be kept in the contract after the second function call, basically making it unrecoverable. Since there are no checks in place to prevent this, this is classified as a high severity since it is an easy mistake to make and leads to loss of funds of the owner.

## Proof of Concept

This can be demonstrated with this test
```javascript
describe('Funds stuck due to wrong order of function calls', async () => {
    it('should trap funds', async () => {
      await deployedFactoryContract.connect(firstAddress).mintReceipt(questId, messageHash, signature)
      await deployedQuestContract.start()
      await ethers.provider.send('evm_increaseTime', [86400])
      await deployedQuestContract.connect(firstAddress).claim()

      await ethers.provider.send('evm_increaseTime', [100001])
      await deployedQuestContract.withdrawFee()
      await deployedQuestContract.withdrawRemainingTokens(owner.address)

      expect(await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)).to.equal(200)
      expect(await deployedSampleErc20Contract.balanceOf(owner.address)).to.be.lessThan(
        totalRewardsPlusFee * 100 - 1 * 1000 - 200
      )
      await ethers.provider.send('evm_increaseTime', [-100001])
      await ethers.provider.send('evm_increaseTime', [-86400])
    })
  })
```
Even though the fee is paid, the contract still retains the fee amount. The owner receives less than the expected amount. This test is a modification of the test `should transfer non-claimable rewards back to owner` already present in `ERC20Quest.spec.ts`.
## Tools Used
Hardhat
## Recommended Mitigation Steps
Only allow fee to be withdrawn after the owner has withdrawn the funds. 
```solidity
// Declare a boolean to check if recovery happened
bool recoveryDone;

function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens) * rewardAmountInWeiOrTokenId;
        uint256 nonClaimableTokens = IERC20(rewardToken).balanceOf(address(this)) - protocolFee() - unclaimedTokens;
        IERC20(rewardToken).safeTransfer(to_, nonClaimableTokens);
   
        // Set recovery bool
        recoveryDone = true;
    }
function withdrawFee() public onlyAdminWithdrawAfterEnd {
        // Check recovery
        require(recoveryDone,"Recover tokens before withdrawing Fees");
        IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
    }
```
