## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-06

# [ETHCrowdfundBase.sol: totalVotingPower is increased too much in the _finalize function](https://github.com/code-423n4/2023-04-party-findings/issues/11) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L273-L292


# Vulnerability details

## Impact
This issue is about how the [`ETHCrowdfundBase._finalize`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L273-L292) functions calls [`PartyGovernanceNFT.increaseTotalVotingPower`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernanceNFT.sol#L193-L197) with an amount that does not reflect the sum of the individual users' voting power.  

Thereby it will become impossible to reach unanimous votes. In other words and more generally the users' votes are worth less than they should be as the percentage is calculated against a total amount that is too big.  

In short, this is how the issue is caused:  
1. The voting power that a user receives is based on the amount they contribute MINUS funding fees
2. The amount of voting power by which `totalVotingPower` is increased is based on the total contributions WITHOUT subtracting funding fees

## Proof of Concept
Let's first look at the affected code and then at the PoC.  

The `votingPower` that a user receives for making a contribution is calculated in the `ETHCrowdfundBase._processContribution` function.  

We can see that first the funding fee is subtracted and then with the lowered `amount`, the `votingPower` is calculated:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L224-L233)  
```solidity
// Subtract fee from contribution amount if applicable.
address payable fundingSplitRecipient_ = fundingSplitRecipient;
uint16 fundingSplitBps_ = fundingSplitBps;
if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
    uint96 feeAmount = (amount * fundingSplitBps_) / 1e4;
    amount -= feeAmount;
}


// Calculate voting power.
votingPower = (amount * exchangeRateBps) / 1e4;
```

Even before that, `totalContributions` has been increased by the full `amount` (funding fees have not been subtracted yet):  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L204-L222)  
```solidity
uint96 newTotalContributions = totalContributions + amount;
uint96 maxTotalContributions_ = maxTotalContributions;
if (newTotalContributions >= maxTotalContributions_) {
    totalContributions = maxTotalContributions_;


    // Finalize the crowdfund.
    // This occurs before refunding excess contribution to act as a
    // reentrancy guard.
    _finalize(maxTotalContributions_);


    // Refund excess contribution.
    uint96 refundAmount = newTotalContributions - maxTotalContributions;
    if (refundAmount > 0) {
        amount -= refundAmount;
        payable(msg.sender).transferEth(refundAmount);
    }
} else {
    totalContributions = newTotalContributions;
}
```
(Note that the above code looks more complicated than it is because it accounts for the fact that `maxTotalContributions` might be reached. But this is not important for explaining this issue)  

When `PartyGovernanceNFT.increaseTotalVotingPower` is called it is with the `newVotingPower` that has been calculated BEFORE funding fees are subtracted:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L278-L288)  
```solidity
uint96 newVotingPower = (totalContributions_ * exchangeRateBps) / 1e4;
party.increaseTotalVotingPower(newVotingPower);


// Transfer fee to recipient if applicable.
address payable fundingSplitRecipient_ = fundingSplitRecipient;
uint16 fundingSplitBps_ = fundingSplitBps;
if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
    uint96 feeAmount = (totalContributions_ * fundingSplitBps_) / 1e4;
    totalContributions_ -= feeAmount;
    fundingSplitRecipient_.transferEth(feeAmount);
}
```

Therefore `totalVotingPower` is increased more than the sum of the voting power that the users have received.  

Let's look at the PoC:  

```javascript
function test_totalVotingPower_increased_too_much() public {
        ReraiseETHCrowdfund crowdfund = _createCrowdfund({
            initialContribution: 0,
            initialContributor: payable(address(0)),
            initialDelegate: address(0),
            minContributions: 0,
            maxContributions: type(uint96).max,
            disableContributingForExistingCard: false,
            minTotalContributions: 2 ether,
            maxTotalContributions: 5 ether,
            duration: 7 days,
            fundingSplitBps: 1000,
            fundingSplitRecipient: payable(address(1))
        });

        address member1 = _randomAddress();
        address member2 = _randomAddress();
        vm.deal(member1, 1 ether);
        vm.deal(member2, 1 ether);

        // Contribute, should be allowed to update delegate
        vm.startPrank(member1);
        crowdfund.contribute{ value: 1 ether }(member1, "");
        vm.stopPrank();

        vm.startPrank(member2);
        crowdfund.contribute{ value: 1 ether }(member2, "");
        vm.stopPrank();

        skip(7 days);
        console.log(party.getGovernanceValues().totalVotingPower);
        crowdfund.finalize();
        console.log(party.getGovernanceValues().totalVotingPower);

        console.log(crowdfund.pendingVotingPower(member1));
        console.log(crowdfund.pendingVotingPower(member2));
    }
```

See that `totalVotingPower` is increased from `0` to `2e18`.  
The voting power of both users is `0.9e18` (10% fee).  

Thereby both users together receive a voting power of `1.8e18` which is only 90% of `2e18`.  

Therefore it is impossible to reach an unanimous vote.  

## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
The fix is easy:  
We must consider the funding fee when increasing the `totalVotingPower`.  

Fix:  
```diff
diff --git a/contracts/crowdfund/ETHCrowdfundBase.sol b/contracts/crowdfund/ETHCrowdfundBase.sol
index 4392655..3c11160 100644
--- a/contracts/crowdfund/ETHCrowdfundBase.sol
+++ b/contracts/crowdfund/ETHCrowdfundBase.sol
@@ -274,10 +274,6 @@ contract ETHCrowdfundBase is Implementation {
         // Finalize the crowdfund.
         delete expiry;
 
-        // Update the party's total voting power.
-        uint96 newVotingPower = (totalContributions_ * exchangeRateBps) / 1e4;
-        party.increaseTotalVotingPower(newVotingPower);
-
         // Transfer fee to recipient if applicable.
         address payable fundingSplitRecipient_ = fundingSplitRecipient;
         uint16 fundingSplitBps_ = fundingSplitBps;
@@ -287,6 +283,10 @@ contract ETHCrowdfundBase is Implementation {
             fundingSplitRecipient_.transferEth(feeAmount);
         }
 
+        // Update the party's total voting power.
+        uint96 newVotingPower = (totalContributions_ * exchangeRateBps) / 1e4;
+        party.increaseTotalVotingPower(newVotingPower);
+        
         // Transfer ETH to the party.
         payable(address(party)).transferEth(totalContributions_);
     }
```


