## Tags

- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- MR-NEW

# [Attacker can cause loss to rToken holders and stakers by running BackingManager._manageTokens before rewards are claimed](https://github.com/code-423n4/2023-02-reserve-mitigation-contest-findings/issues/22) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/BackingManager.sol#L105-L153


# Vulnerability details

## Impact
The assets that back the rTokens are held by the `BackingManager` and can earn rewards.  

The rewards can be claimed via the [`TradingP1.claimRewards`](https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/mixins/Trading.sol#L82-L84) and [`TradingP1.claimRewardsSingle`](https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/mixins/Trading.sol#L90-L92) function.  

The `BackingManager` inherits from `TradingP1` and therefore the above functions can be used to claim rewards.  

The issue is that the `BackingManager` does not claim rewards as part of its [`_manageTokens`](https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/BackingManager.sol#L105-L153) function.  

So recollateralization can occur before rewards have been claimed.  

There exist possibilities how an attacker can exploit this to cause a loss to rToken holders and rsr stakers.  

## Proof of Concept
Let's think about an example for such a scenario:  

Assume that the `RToken` is backed by a considerable amount of `TokenA`

`TokenA` earns rewards but not continuously. Bigger amounts of rewards are paid out periodically. Say 5% rewards every year.  

Assume further that the `RToken` is currently undercollateralized.  

The attacker can now front-run the claiming of rewards and perform recollateralization.  

The recollateralization might now seize `rsr` from stakers or take an unnecessary haircut.  

## Tools Used
VSCode

## Recommended Mitigation Steps
I suggest that the `BackingManager._manageTokens` function calls `claimRewards`. Even before it calculates how many baskets it holds:  

```diff
diff --git a/contracts/p1/BackingManager.sol b/contracts/p1/BackingManager.sol
index fc38ce29..e17bb63d 100644
--- a/contracts/p1/BackingManager.sol
+++ b/contracts/p1/BackingManager.sol
@@ -113,6 +113,8 @@ contract BackingManagerP1 is TradingP1, IBackingManager {
         uint48 basketTimestamp = basketHandler.timestamp();
         if (block.timestamp < basketTimestamp + tradingDelay) return;
 
+        this.claimRewards();
+
         uint192 basketsHeld = basketHandler.basketsHeldBy(address(this));
 
         // if (basketHandler.fullyCollateralized())
```
