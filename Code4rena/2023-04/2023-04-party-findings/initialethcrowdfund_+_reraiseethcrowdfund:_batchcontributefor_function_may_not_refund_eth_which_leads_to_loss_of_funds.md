## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- upgraded by judge
- edited-by-warden
- H-07

# [InitialETHCrowdfund + ReraiseETHCrowdfund: batchContributeFor function may not refund ETH which leads to loss of funds](https://github.com/code-423n4/2023-04-party-findings/issues/7) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L235-L268
https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L174-L202


# Vulnerability details

## Impact
This vulnerability exists in both the [`InitialETHCrowdfund`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/InitialETHCrowdfund.sol) and [`ReraiseETHCrowdfund`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol) contracts in exactly the same way.  

I will continue this report by explaining the issue in only one contract. The mitigation section however contains the fix for both instances.  

The [`batchContributeFor`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L235-L268) function is a wrapper that allows to make multiple calls to `contributeFor` within one function call.  

It is possible to specify that this function should not revert when one individual call to `contributeFor` fails by setting `args.revertOnFailure=false`.  

The issue is that in this case the ETH for a failed contribution is not refunded which leads a loss of funds for the user calling the function.  

Note:  
This issue also exists in the [`Crowdfund.batchContributeFor`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/Crowdfund.sol#L367-L385) function which is out of scope. The sponsor knows about this and will fix it.  

## Proof of Concept
Let's look at the `batchContributeFor` function:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L235-L268)  
```solidity
function batchContributeFor(
    BatchContributeForArgs calldata args
) external payable onlyDelegateCall returns (uint96[] memory votingPowers) {
    uint256 numContributions = args.recipients.length;
    votingPowers = new uint96[](numContributions);


    uint256 ethAvailable = msg.value;
    for (uint256 i; i < numContributions; ++i) {
        ethAvailable -= args.values[i];


        (bool s, bytes memory r) = address(this).call{ value: args.values[i] }(
            abi.encodeCall(
                this.contributeFor,
                (
                    args.tokenIds[i],
                    args.recipients[i],
                    args.initialDelegates[i],
                    args.gateDatas[i]
                )
            )
        );


        if (!s) {
            if (args.revertOnFailure) {
                r.rawRevert();
            }
        } else {
            votingPowers[i] = abi.decode(r, (uint96));
        }
    }


    // Refund any unused ETH.
    if (ethAvailable > 0) payable(msg.sender).transfer(ethAvailable);
}
```

We can see that `ethAvailable` is reduced before every call to `contributeFor`:  

```solidity
ethAvailable -= args.values[i];
```

But it is only checked later if the call was successful:  

```solidity
if (!s) {
    if (args.revertOnFailure) {
        r.rawRevert();
    }
```

And if `args.revertOnFailure=false` there is no revert and `ethAvailable` is not increased again.  

Therefore the user has to pay for failed contributions.  

Add the following test to the `InitialETHCrowdfund.t.sol` test file:  

```solidity
function test_batchContributeFor_noETHRefund() public {
    InitialETHCrowdfund crowdfund = _createCrowdfund({
        initialContribution: 0,
        initialContributor: payable(address(0)),
        initialDelegate: address(0),
        minContributions: 1 ether,
        maxContributions: type(uint96).max,
        disableContributingForExistingCard: false,
        minTotalContributions: 3 ether,
        maxTotalContributions: 5 ether,
        duration: 7 days,
        fundingSplitBps: 0,
        fundingSplitRecipient: payable(address(0))
    });
    Party party = crowdfund.party();

    address sender = _randomAddress();
    vm.deal(sender, 2.5 ether);

    // Batch contribute for
    vm.prank(sender);
    uint256[] memory tokenIds = new uint256[](3);
    address payable[] memory recipients = new address payable[](3);
    address[] memory delegates = new address[](3);
    uint96[] memory values = new uint96[](3);
    bytes[] memory gateDatas = new bytes[](3);
    for (uint256 i; i < 3; ++i) {
        recipients[i] = _randomAddress();
        delegates[i] = _randomAddress();
        values[i] = 1 ether;
    }

    // @audit-info set values[2] = 0.5 ether such that contribution fails (minContribution = 1 ether)
    values[2] = 0.5 ether;

    uint96[] memory votingPowers = crowdfund.batchContributeFor{ value: 2.5 ether }(
        InitialETHCrowdfund.BatchContributeForArgs({
            tokenIds: tokenIds,
            recipients: recipients,
            initialDelegates: delegates,
            values: values,
            gateDatas: gateDatas,
            revertOnFailure: false
        })
    );

    // @audit-info balance of sender is 0 ETH even though 0.5 ETH of the 2.5 ETH should have been refunded
    assertEq(address(sender).balance, 0 ether);
}
```

The `sender` sends 2.5 ETH and 1 of the 3 contributions fails since `minContribution` is above the amount the `sender` wants to contribute (Note that in practice there are more ways for the contribution to fail).  

The sender's balance in the end is 0 ETH which shows that there is no refund.  

## Tools Used
VSCode,Foundry

## Recommended Mitigation Steps
The following changes need to be made to the `InitialETHCrowdfund` and `ReraiseETHCrowdfund` contracts:  

```diff
diff --git a/contracts/crowdfund/InitialETHCrowdfund.sol b/contracts/crowdfund/InitialETHCrowdfund.sol
index 8ab3b5c..19e09ac 100644
--- a/contracts/crowdfund/InitialETHCrowdfund.sol
+++ b/contracts/crowdfund/InitialETHCrowdfund.sol
@@ -240,8 +240,6 @@ contract InitialETHCrowdfund is ETHCrowdfundBase {
 
         uint256 ethAvailable = msg.value;
         for (uint256 i; i < numContributions; ++i) {
-            ethAvailable -= args.values[i];
-
             (bool s, bytes memory r) = address(this).call{ value: args.values[i] }(
                 abi.encodeCall(
                     this.contributeFor,
@@ -260,6 +258,7 @@ contract InitialETHCrowdfund is ETHCrowdfundBase {
                 }
             } else {
                 votingPowers[i] = abi.decode(r, (uint96));
+                ethAvailable -= args.values[i];
             }
         }
```

```diff
diff --git a/contracts/crowdfund/ReraiseETHCrowdfund.sol b/contracts/crowdfund/ReraiseETHCrowdfund.sol
index 580623d..ad70b27 100644
--- a/contracts/crowdfund/ReraiseETHCrowdfund.sol
+++ b/contracts/crowdfund/ReraiseETHCrowdfund.sol
@@ -179,8 +179,6 @@ contract ReraiseETHCrowdfund is ETHCrowdfundBase, CrowdfundNFT {
 
         uint256 ethAvailable = msg.value;
         for (uint256 i; i < numContributions; ++i) {
-            ethAvailable -= args.values[i];
-
             (bool s, bytes memory r) = address(this).call{ value: args.values[i] }(
                 abi.encodeCall(
                     this.contributeFor,
@@ -194,6 +192,7 @@ contract ReraiseETHCrowdfund is ETHCrowdfundBase, CrowdfundNFT {
                 }
             } else {
                 votingPowers[i] = abi.decode(r, (uint96));
+                ethAvailable -= args.values[i];
             }
         }
```

Now `ethAvailable` is only reduced when the call to `contributeFor` was successful.  
