## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-10

# [OperatorProposal.sol: Leftover ETH is not refunded to the msg.sender](https://github.com/code-423n4/2023-04-party-findings/issues/5) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/OperatorProposal.sol#L25-L49


# Vulnerability details

## Impact
The [`OperatorProposal`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/OperatorProposal.sol#L7-L63) contract is a type of proposal that allows to execute operations on contracts that implement the [`IOperator`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/operators/IOperator.sol) interface.  

Upon execution of the proposal it might be necessary that the `executor` provides ETH.  

This is true especially when `allowOperatorsToSpendPartyEth=false`, i.e. when ETH cannot be spent from the Party's balance. So it must be provided by the `executor`.  

The amount of ETH that is needed to execute the operation is sent to the operator contract:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/OperatorProposal.sol#L45)  
```solidity
data.operator.execute{ value: data.operatorValue }(data.operatorData, executionData);
```

The operator contract then spends whatever amount of ETH is actually necessary and returns the remaining ETH.  

For example the [`CollectionBatchBuyOperator`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L14-L224) contract may not spend all of the ETH because the actual purchases that are made are not necessarily known at the time the proposal is created. Also not all purchases may succeed.  

So it is clear that some of the ETH may be returned from the `operator` to the `OperatorProposal` contract.  

The issue is that the remaining ETH is not refunded to the `executor` and therefore this results in a direct loss of funds for the `executor`.  

I discussed this issue with the sponsor and it is clear that the remaining ETH needs to be refunded when `allowOperatorsToSpendPartyEth=false`.  

However it is not clear what to do when `allowOperatorsToSpendPartyEth=true`. In this case ETH can be spent from the party's balance. So there should be limited use cases for the `executor` providing additional ETH.  

But if the `executor` provides additional ETH what should happen?  
Should the ETH be taken from the `executor` first? Or should it be taken from the Party balance first?  

The sponsor mentioned that since there are limited use cases for the `executor` providing additional ETH it may be ok to not refund ETH at all.  

I disagree with this. Even when `allowOperatorsToSpendPartyEth=true` there should be a policy for refunds. I.e. the necessary ETH should either be taken from the Party's balance or from the `executor` first and any remaining funds from the `executor` should be returned.  
However since it is not clear how to proceed in this case and since it is less important compared to the case where `allowOperatorsToSpendPartyEth=false` I will only make a suggestion for the case where `allowOperatorsToSpendPartyEth=false`.  

The sponsor should decide what to do in the other case and make the appropriate changes.  

## Proof of Concept
When the `executor` executes an `OperatorProposal`, `operatorValue` amount of ETH is sent to the `operator` contract (when `allowOperatorsToSpendPartyEth=false` all of these funds must come from the `msg.value`):  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/OperatorProposal.sol#L40-L45)  
```solidity
if (!allowOperatorsToSpendPartyEth && data.operatorValue > msg.value) {
    revert NotEnoughEthError(data.operatorValue, msg.value);
}


// Execute the operation.
data.operator.execute{ value: data.operatorValue }(data.operatorData, executionData);
```

Currently the only `operator` contract that is implemented is the `CollectionBatchBuyOperator` and as explained above not all of the funds may be used so the funds are sent back to the Party:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L191-L192)  
```solidity
uint256 unusedEth = msg.value - totalEthUsed;
if (unusedEth > 0) payable(msg.sender).transferEth(unusedEth);
```

However after calling the `operator` contract, the `OperatorProposal` contract just returns without sending back the unused funds to the `executor` (`msg.sender`).  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/OperatorProposal.sol#L47-L48)  
```solidity
// Nothing left to do.
return "";
```

So there is a loss of funds for the `executor`. The leftover funds are effectively transferred to the Party.  

## Tools Used
VSCode


## Recommended Mitigation Steps
As mentioned before, this is only a fix for the case when `allowOperatorsToSpendPartyEth=false`.  

Fix:  
```diff
diff --git a/contracts/proposals/OperatorProposal.sol b/contracts/proposals/OperatorProposal.sol
index 23e2897..507e0d5 100644
--- a/contracts/proposals/OperatorProposal.sol
+++ b/contracts/proposals/OperatorProposal.sol
@@ -4,7 +4,11 @@ pragma solidity 0.8.17;
 import "./IProposalExecutionEngine.sol";
 import "../operators/IOperator.sol";
 
+import "../utils/LibAddress.sol";
+
 contract OperatorProposal {
+    using LibAddress for address payable;
+    
     struct OperatorProposalData {
         // Addresses that are allowed to execute the proposal and decide what
         // calldata used by the operator proposal at the time of execution.
@@ -41,9 +45,17 @@ contract OperatorProposal {
             revert NotEnoughEthError(data.operatorValue, msg.value);
         }
 
+        uint256 partyBalanceBefore = address(this).balance - msg.value;
+
         // Execute the operation.
         data.operator.execute{ value: data.operatorValue }(data.operatorData, executionData);
 
+        if (!allowOperatorsToSpendPartyEth) {
+            if (address(this).balance - partyBalanceBefore > 0) {
+                payable(msg.sender).transferEth(address(this).balance - partyBalanceBefore);
+            }
+        }
+
         // Nothing left to do.
         return "";
     }
```

