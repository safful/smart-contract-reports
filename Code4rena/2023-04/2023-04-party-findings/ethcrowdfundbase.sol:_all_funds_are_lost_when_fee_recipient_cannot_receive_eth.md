## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-08

# [ETHCrowdfundBase.sol: all funds are lost when fee recipient cannot receive ETH](https://github.com/code-423n4/2023-04-party-findings/issues/8) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L273-L292


# Vulnerability details

## Impact
In the [`ETHCrowdfundBase`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ETHCrowdfundBase.sol) contract a [`fundingSplitRecipient`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L142) address is configured which receives a percentage of the funds in case the crowdfund is won.  

Neither the `fundingSplitRecipient` address nor the `fundingSplitBps` percentage can be changed.  

The issue is that the `_finalize` function can only succeed when the fees can be transferred to the recipient.  

However the recipient contract may revert when it receives ETH. This causes all ETH in the `ETHCrowdfundBase` contract to be stuck.  

## Proof of Concept
When the crowdfund is won the `finalize` function needs to be called which calls `_finalize`:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L273-L292)  
```solidity
function _finalize(uint96 totalContributions_) internal {
    // Finalize the crowdfund.
    delete expiry;


    // Update the party's total voting power.
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


    // Transfer ETH to the party.
    payable(address(party)).transferEth(totalContributions_);
}
```

Here you can see that the `feeAmount` is transferred to the `fundingSplitRecipient`:  

```solidity
fundingSplitRecipient_.transferEth(feeAmount);
```

If the recipient contract reverts, the ETH cannot be transferred and the crowdfund cannot be finalized.  

But the users can also not get a refund because the crowdfund is in the `Won` state. So there is no way to get the funds out of the contract which means they are lost. Also the users don't get the voting power that they are supposed to get from the crowdfund.  

This could be used in a griefing attack where the `fundingSplitRecipient` is set such that it can be made to revert.  

Users that fall into this "trap" will lose all their funds. It can also just happen by mistake that a bad `fundingSplitRecipient` is set.  

## Tools Used
VSCode

## Recommended Mitigation Steps
I recommend to pay the fees in a separate function such that it is separated from the `_finalize` function.  


```diff
diff --git a/contracts/crowdfund/ETHCrowdfundBase.sol b/contracts/crowdfund/ETHCrowdfundBase.sol
index 4392655..5f68406 100644
--- a/contracts/crowdfund/ETHCrowdfundBase.sol
+++ b/contracts/crowdfund/ETHCrowdfundBase.sol
@@ -62,6 +62,8 @@ contract ETHCrowdfundBase is Implementation {
     error BelowMinimumContributionsError(uint96 contributions, uint96 minContributions);
     error AboveMaximumContributionsError(uint96 contributions, uint96 maxContributions);
     error ContributingForExistingCardDisabledError();
+    error NotFinalizedError();
+    error FundingFeesAlreadyPaidError();
 
     event Contributed(
         address indexed sender,
@@ -109,6 +111,8 @@ contract ETHCrowdfundBase is Implementation {
     /// @notice The address a contributor is delegating their voting power to.
     mapping(address => address) public delegationsByContributor;
 
+    bool public fundingFeesPaid;
+
     // Initialize storage for proxy contracts, credit initial contribution (if
     // any), and setup gatekeeper.
     function _initialize(ETHCrowdfundOptions memory opts) internal {
@@ -278,7 +282,20 @@ contract ETHCrowdfundBase is Implementation {
         uint96 newVotingPower = (totalContributions_ * exchangeRateBps) / 1e4;
         party.increaseTotalVotingPower(newVotingPower);
 
+        // Transfer ETH to the party.
+        payable(address(party)).transferEth(totalContributions_);
+    }
+
+    function sendFundingFees() external {
+        CrowdfundLifecycle lc = getCrowdfundLifecycle();
+        
+        if (lc != CrowdfundLifecycle.Finalized) revert NotFinalizedError();
+        if (fundingFeesPaid) revert FundingFeesAlreadyPaidError();
+
+        fundingFeesPaid = true;
+
         // Transfer fee to recipient if applicable.
+        uint96 totalContributions_ = totalContributions;
         address payable fundingSplitRecipient_ = fundingSplitRecipient;
         uint16 fundingSplitBps_ = fundingSplitBps;
         if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
@@ -286,8 +303,5 @@ contract ETHCrowdfundBase is Implementation {
             totalContributions_ -= feeAmount;
             fundingSplitRecipient_.transferEth(feeAmount);
         }
-
-        // Transfer ETH to the party.
-        payable(address(party)).transferEth(totalContributions_);
     }
 }
```

Alternatively it may also be an option to just send the fees to the party in case the transfer to the recipient fails.  
