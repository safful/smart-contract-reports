## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- upgraded by judge
- edited-by-warden
- H-08

# [Claiming accumulated rewards while the contract is underfunded can lead to a loss of rewards](https://github.com/code-423n4/2023-05-ajna-findings/issues/251) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L815


# Vulnerability details

The claimable rewards for an NFT staker are capped at the Ajna token balance at the time of claiming, which can lead to a loss of rewards if the `RewardsManager` contract is underfunded with Ajna tokens.

## Impact

Loss of rewards if the `RewardsManager` contract is underfunded with Ajna tokens.

## Proof of Concept

The `RewardsManager` contract keeps track of the rewards earned by an NFT staker. The accumulated rewards are claimed by calling the `RewardsManager.claimRewards` function. Internally, the `RewardsManager._claimRewards` function transfers the accumulated rewards to the staker.

However, the transferrable amount of Ajna token rewards are capped at the Ajna token balance at the time of claiming. If the accumulated rewards are higher than the Ajna token balance, the claimer will receive fewer rewards than expected. The remaining rewards cannot be claimed at a later time as the `RewardsManager` contract does not keep track of the rewards that were not transferred.

### Note

This issue was already reported on [Sherlock's audit contest](https://github.com/sherlock-audit/2023-01-ajna-judging/issues/120), and was marked as [`Fixed`](https://github.com/ajna-finance/audits) by the Ajna team (Issue M-8). 

Nevertheless, the problem still exists, as it can be seen through the following test:

```diff
diff --git a/ajna-core/src/RewardsManager.sol b/ajna-core/src/RewardsManager.sol
index 314b476..6642a4e 100644
--- a/ajna-core/src/RewardsManager.sol
+++ b/ajna-core/src/RewardsManager.sol
@@ -582,6 +582,7 @@ contract RewardsManager is IRewardsManager, ReentrancyGuard {
             epochToClaim_
         );
 
+        // @audit-issue rewardsEarned (ClaimReward event) is not necessarily what's transferred (Transfer event)
         emit ClaimRewards(
             msg.sender,
             ajnaPool_,
@@ -812,6 +813,7 @@ contract RewardsManager is IRewardsManager, ReentrancyGuard {
         // check that rewards earned isn't greater than remaining balance
         // if remaining balance is greater, set to remaining balance
         uint256 ajnaBalance = IERC20(ajnaToken).balanceOf(address(this));
+        // @audit-issue rewardsEarned (ClaimReward event) is not necessarily what's transferred (Transfer event)
         if (rewardsEarned_ > ajnaBalance) rewardsEarned_ = ajnaBalance;
 
         if (rewardsEarned_ != 0) {
diff --git a/ajna-core/tests/forge/unit/Rewards/RewardsDSTestPlus.sol b/ajna-core/tests/forge/unit/Rewards/RewardsDSTestPlus.sol
index 93fe062..74a70d5 100644
--- a/ajna-core/tests/forge/unit/Rewards/RewardsDSTestPlus.sol
+++ b/ajna-core/tests/forge/unit/Rewards/RewardsDSTestPlus.sol
@@ -162,6 +162,8 @@ abstract contract RewardsDSTestPlus is IRewardsManagerEvents, ERC20HelperContrac
         uint256 currentBurnEpoch = IPool(pool).currentBurnEpoch();
         vm.expectEmit(true, true, true, true);
         emit ClaimRewards(from, pool, tokenId, epochsClaimed, reward);
+        vm.expectEmit(true, true, true, true);
+        emit Transfer(address(_rewardsManager), from, reward);
         _rewardsManager.claimRewards(tokenId, currentBurnEpoch);
 
         assertEq(_ajnaToken.balanceOf(from), fromAjnaBal + reward);
@@ -267,8 +269,8 @@ abstract contract RewardsHelperContract is RewardsDSTestPlus {
         _poolTwo       = ERC20Pool(_poolFactory.deployPool(address(_collateralTwo), address(_quoteTwo), 0.05 * 10**18));
 
         // provide initial ajna tokens to staking rewards contract
-        deal(_ajna, address(_rewardsManager), 100_000_000 * 1e18);
-        assertEq(_ajnaToken.balanceOf(address(_rewardsManager)), 100_000_000 * 1e18);
+        deal(_ajna, address(_rewardsManager), 40 * 1e18);
+        assertEq(_ajnaToken.balanceOf(address(_rewardsManager)), 40 * 1e18); // @audit-issue RewardsManager is now underfunded, contains less AJNA than users' rewards
     }
 
     // create a new test borrower with quote and collateral sufficient to draw a specified amount of debt
diff --git a/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol b/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
index 4100e9f..3eaacd7 100644
--- a/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
+++ b/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
@@ -1843,6 +1843,15 @@ contract RewardsManagerTest is RewardsHelperContract {
         });
         assertLt(_ajnaToken.balanceOf(_minterOne), tokensToBurn);
 
+
+        // try to claim again and get remaining rewards, will revert with `AlreadyClaimed()`
+        _claimRewards({
+            pool:          address(_pool),
+            from:          _minterOne,
+            tokenId:       tokenIdOne,
+            reward:        40.899689081331351737 * 1e18,
+            epochsClaimed: _epochsClaimedArray(1, 0)
+        });
     }
 
     /********************/

```

Since the problem still exists, I am reporting it here. You can find below a conversation with Ian Harvey from the Ajna team, where we discuss how the problem was incorrectly marked as solved:

> aviggiano — Yesterday at 4:37 PM
> Hi there
> I am reviewing the Ajna smart contracts and I have a question regarding previous audit reports. 
> https://github.com/ajna-finance/audits
> It seems like some findings are marked as "Fixed" but I believe they were not (see M-8). 
> Should I re-submit a previous finding, if the contract is in scope? or are those considered out of scope?
> M-8 is this one
> https://github.com/sherlock-audit/2023-01-ajna-judging/issues/120
> Ian Harvey | Ajna — Yesterday at 7:01 PM
> Checking
> Ian Harvey | Ajna — Yesterday at 7:11 PM
> That was solved here -> https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L811

## Tools Used

Past audit report

## Recommended Mitigation Steps

- Consider reverting if there are insufficient Ajna tokens available as rewards. This is the best immediate solution to the problem.
- Create unit tests for each issue identified in the audit report and confirm that it has been properly addressed. This will prevent recurring problems where the development team believes an issue has been resolved, but in reality, it has not.
- Create a separate pull request for each finding and mark the issue in the audit table. This will help developers and auditors verify whether the issue has been resolved or not, and will make future audits more manageable, ultimately improving the overall quality and security of the protocol.











## Assessed type

Math