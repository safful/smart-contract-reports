## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-09

# [InitialETHCrowdfund + ReraiseETHCrowdfund: Gatekeeper checks wrong address](https://github.com/code-423n4/2023-04-party-findings/issues/6) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L282-L293
https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L215-L226


# Vulnerability details

## Impact
This vulnerability exists in both the [`InitialETHCrowdfund`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/InitialETHCrowdfund.sol) and [`ReraiseETHCrowdfund`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol) contracts in exactly the same way.  

I will continue this report by explaining the issue in only one contract. The mitigation section however contains the fix for both instances.  

When making a contribution there is a check with the `gatekeeper` (if it is configured):  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L282-L293)  
```solidity
// Must not be blocked by gatekeeper.
IGateKeeper _gateKeeper = gateKeeper;
if (_gateKeeper != IGateKeeper(address(0))) {
    if (!_gateKeeper.isAllowed(contributor, gateKeeperId, gateData)) {
        revert NotAllowedByGateKeeperError(
            contributor,
            _gateKeeper,
            gateKeeperId,
            gateData
        );
    }
}
```

The issue is that the first argument to the `isAllowed` function is wrong. It is `contributor` but it should be `msg.sender`.  

The impact of this is that it will be possible for unauthorized users to make contributions.  

## Proof of Concept
Fortunately the new `InitialETHCrowdfund` and `ReraiseETHCrowdfund` contracts are very similar to the already audited other crowdfund contracts.  

So we can look into the `Crowdfund.sol` code and see how the `gatekeeper.isAllowed` function should be called:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/Crowdfund.sol#L620-L644)  
```solidity
    function _contribute(
        address contributor,
        address delegate,
        uint96 amount,
        uint96 previousTotalContributions,
        bytes memory gateData
    ) private {
        if (contributor == address(this)) revert InvalidContributorError();


        if (amount == 0) return;


        // Must not be blocked by gatekeeper.
        {
            IGateKeeper _gateKeeper = gateKeeper;
            if (_gateKeeper != IGateKeeper(address(0))) {
                if (!_gateKeeper.isAllowed(msg.sender, gateKeeperId, gateData)) {
                    revert NotAllowedByGateKeeperError(
                        msg.sender,
                        _gateKeeper,
                        gateKeeperId,
                        gateData
                    );
                }
            }
        }
```

We can see that the first argument to the `gatekeeper.isAllowed` function is `msg.sender`.  

This means that when User A contributes for User B, the address that is checked is the address of User A and not the address of User B.  

The new crowdfund contracts however check `contributor`:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/InitialETHCrowdfund.sol#L282-L293)  
```solidity
// Must not be blocked by gatekeeper.
IGateKeeper _gateKeeper = gateKeeper;
if (_gateKeeper != IGateKeeper(address(0))) {
    if (!_gateKeeper.isAllowed(contributor, gateKeeperId, gateData)) {
        revert NotAllowedByGateKeeperError(
            contributor,
            _gateKeeper,
            gateKeeperId,
            gateData
        );
    }
}
```

This means that when User A contributes for User B, the address of User B is checked. However it should be the address of User A (as seen above).  

Imagine a situation where three addresses are whitelisted by the gatekeeper:  

```
Alice
Bob
Chris
```

What should be checked by the gatekeeper is that only Alice, Bob and Chris can make contributions but they should be able to make contributions for everyone (check `msg.sender` instead of contributor).  

What is actually checked is that any user can make a contribution but they can only contribute to Alice, Bob and Chris (check contributor instead of `msg.sender`).  



## Tools Used
VSCode

## Recommended Mitigation Steps
In both contracts the `msg.sender` needs to be checked instead of `contributor`.  

```diff
diff --git a/contracts/crowdfund/InitialETHCrowdfund.sol b/contracts/crowdfund/InitialETHCrowdfund.sol
index 8ab3b5c..fa8ec5d 100644
--- a/contracts/crowdfund/InitialETHCrowdfund.sol
+++ b/contracts/crowdfund/InitialETHCrowdfund.sol
@@ -282,9 +282,9 @@ contract InitialETHCrowdfund is ETHCrowdfundBase {
         // Must not be blocked by gatekeeper.
         IGateKeeper _gateKeeper = gateKeeper;
         if (_gateKeeper != IGateKeeper(address(0))) {
-            if (!_gateKeeper.isAllowed(contributor, gateKeeperId, gateData)) {
+            if (!_gateKeeper.isAllowed(msg.sender, gateKeeperId, gateData)) {
                 revert NotAllowedByGateKeeperError(
-                    contributor,
+                    msg.sender,
                     _gateKeeper,
                     gateKeeperId,
                     gateData
```

```diff
diff --git a/contracts/crowdfund/ReraiseETHCrowdfund.sol b/contracts/crowdfund/ReraiseETHCrowdfund.sol
index 580623d..72f3a20 100644
--- a/contracts/crowdfund/ReraiseETHCrowdfund.sol
+++ b/contracts/crowdfund/ReraiseETHCrowdfund.sol
@@ -215,9 +215,9 @@ contract ReraiseETHCrowdfund is ETHCrowdfundBase, CrowdfundNFT {
         // Must not be blocked by gatekeeper.
         IGateKeeper _gateKeeper = gateKeeper;
         if (_gateKeeper != IGateKeeper(address(0))) {
-            if (!_gateKeeper.isAllowed(contributor, gateKeeperId, gateData)) {
+            if (!_gateKeeper.isAllowed(msg.sender, gateKeeperId, gateData)) {
                 revert NotAllowedByGateKeeperError(
-                    contributor,
+                    msg.sender,
                     _gateKeeper,
                     gateKeeperId,
                     gateData
```

On this note it is important to mention that there is also an issue in `Crowdfund.sol` which is out of scope but the issue is of importance here:  

The issue is in the `Crowdfund.batchContributeFor` function.  
The function calls `this.contributeFor` [Link](https://github.com/PartyDAO/party-protocol/blob/3313c24c85d7429346af939897c19deeef7952f5/contracts/crowdfund/Crowdfund.sol#L365-L368).  

So when the call is made, `msg.sender` is the address of the crowdfund and not the address of the user.  

Therefore the gatekeeper check is wrong [Link](https://github.com/PartyDAO/party-protocol/blob/3313c24c85d7429346af939897c19deeef7952f5/contracts/crowdfund/Crowdfund.sol#L596).  

This is clearly not how the gatekeeper should be used. The gatekeeper should check the address of the user.  

If you implement in the `ReraiseETHCrowdfund` and `InitialETHCrowdfund` contracts the changes I suggested, the same issue will be introduced there.  

The solution is to call `_contributeFor` directly and to remove the `revertOnFailure` option. Or do a more involved change with supplying the correct `msg.sender`.  


