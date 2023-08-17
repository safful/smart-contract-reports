## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-19

# [In case that `unstakingDelay` is decreased, users who have previously unstaked would have to wait more than `unstakingDelay` for new unstakes](https://github.com/code-423n4/2023-01-reserve-findings/issues/210) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/StRSR.sol#L560-L564


# Vulnerability details

## Impact
Users who wish to unstake their RSR from StRSR have to first unstake and then wait `unstakingDelay` till they can actually withdraw their stake.
The `unstakingDelay` can change by the governance.
The issue is that when the `unstakingDelay` is decreased - users that have pending unstakes (aka drafts) would have to wait till the old delay has passed for the pending draft (not only for their pending drafts, but also for any new draft they wish to create. e.g. if the unstaking delay was 6 months and was changed to 2 weeks, if a user has a pending draft that was created a month before the change the user would have to wait at least 5 months since the change for every new draft).

## Proof of Concept
The following PoC shows an example similar to above:
* Unstaking delay was 6 months
* Bob unstaked (create a draft) 1 wei of RSR
* Unstaking delay was changed to 2 weeks
* Both Bob and Alice unstake their remaining stake
* Alice can withdraw her stake after 2 weeks
* Bob has to wait 6 months in order to withdraw both that 1 wei and the remaining of the stake


```diff
diff --git a/test/ZZStRSR.test.ts b/test/ZZStRSR.test.ts
index f507cd50..3312686a 100644
--- a/test/ZZStRSR.test.ts
+++ b/test/ZZStRSR.test.ts
@@ -599,6 +599,8 @@ describe(`StRSRP${IMPLEMENTATION} contract`, () => {
       let amount2: BigNumber
       let amount3: BigNumber
 
+      let sixMonths = bn(60*60*24*30*6);
+
       beforeEach(async () => {
         stkWithdrawalDelay = bn(await stRSR.unstakingDelay()).toNumber()
 
@@ -608,18 +610,56 @@ describe(`StRSRP${IMPLEMENTATION} contract`, () => {
         amount3 = bn('3e18')
 
         // Approve transfers
-        await rsr.connect(addr1).approve(stRSR.address, amount1)
+        await rsr.connect(addr1).approve(stRSR.address, amount1.add(1))
         await rsr.connect(addr2).approve(stRSR.address, amount2.add(amount3))
 
+        
         // Stake
-        await stRSR.connect(addr1).stake(amount1)
+        await stRSR.connect(addr1).stake(amount1.add(1))
         await stRSR.connect(addr2).stake(amount2)
         await stRSR.connect(addr2).stake(amount3)
 
-        // Unstake - Create withdrawal
-        await stRSR.connect(addr1).unstake(amount1)
+        // here
+        let sixMonths = bn(60*60*24*30*6);
+        // gov thinks it's a good idea to set delay to 6 months
+        await expect(stRSR.connect(owner).setUnstakingDelay(sixMonths))
+        .to.emit(stRSR, 'UnstakingDelaySet')
+        .withArgs(config.unstakingDelay, sixMonths);
+
+        // Poor Bob created a draft when unstaking delay was 6 months
+        await stRSR.connect(addr1).unstake(bn(1))
+
+        // gov revise their previous decision and set unstaking delay back to 2 weeks
+        await expect(stRSR.connect(owner).setUnstakingDelay(config.unstakingDelay))
+        .to.emit(stRSR, 'UnstakingDelaySet')
+        .withArgs(sixMonths, config.unstakingDelay);
+
+        // now both Bob and Alice decide to unstake
+        await stRSR.connect(addr1).unstake(amount1);
+        await stRSR.connect(addr2).unstake(amount2);
+
+      })
+
+      it('PoC user 1 can\'t withdraw', async () => {
+        // Get current balance for user
+        const prevAddr1Balance = await rsr.balanceOf(addr1.address)
+
+        // 6 weeks have passed, much more than current delay
+        await advanceTime(stkWithdrawalDelay * 3)
+
+
+        // Alice can happily withdraw her stake
+        await stRSR.connect(addr2).withdraw(addr2.address, 1)
+        // Bob can't withdraw his stake and has to wait 6 months
+        // Bob is now very angry and wants to talk to the manager
+        await expect(stRSR.connect(addr1).withdraw(addr1.address, 2)).to.be.revertedWith(
+          'withdrawal unavailable'
+        )
+
+
       })
 
+      return; // don't run further test
       it('Should revert withdraw if Main is paused', async () => {
         // Get current balance for user
         const prevAddr1Balance = await rsr.balanceOf(addr1.address)
@@ -1027,6 +1067,7 @@ describe(`StRSRP${IMPLEMENTATION} contract`, () => {
       })
     })
   })
+  return; // don't run further test
 
   describe('Add RSR / Rewards', () => {
     const initialRate = fp('1')
```


## Recommended Mitigation Steps
Allow users to use current delay even if it was previously higher. I think this should apply not only to new drafts but also for drafts that were created before the change.
Alternatively, the protocol can set a rule that even if the staking delay was lowered stakers have to wait at least the old delay since the change till they can withdraw.  But in this case the rule should apply to everybody regardless if they have pending drafts or not.