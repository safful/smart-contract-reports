## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [_updateBucketExchangeRateAndCalculateRewards reward calculation accuracy loss](https://github.com/code-423n4/2023-05-ajna-findings/issues/394) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L792-L801


# Vulnerability details

## Impact

Divide first and then multiply, which has precision loss. The loss depends on the denominator, which is at most `denominator - 1`.   
Although `Maths.wdiv rounded` adopted in _updateBucketExchangeRateAndCalculateRewards, reduce the loss, but theoretically `interestFactor` loss is about `interestEarned_ / 2`.         
This means that the more interest are earned, the more users lose.   

## Proof of Concept

Modify for testing

```diff
diff --git a/ajna-core/src/RewardsManager.sol b/ajna-core/src/RewardsManager.sol
index 314b476..421940f 100644
--- a/ajna-core/src/RewardsManager.sol
+++ b/ajna-core/src/RewardsManager.sol
@@ -801,8 +801,12 @@ contract RewardsManager is IRewardsManager, ReentrancyGuard {
                 rewards_ += Maths.wmul(UPDATE_CLAIM_REWARD, Maths.wmul(burnFactor, interestFactor));
             }
         }
+
+        emit CalculateReward(rewards_);
     }
 
+    event CalculateReward(uint256);
+
     /** @notice Utility method to transfer `Ajna` rewards to the sender
      *  @dev   This method is used to transfer rewards to the `msg.sender` after a successful claim or update.
      *  @dev   It is used to ensure that rewards claimers will be able to claim some portion of the remaining tokens if a claim would exceed the remaining contract balance.
```

Modify reward calculation

```diff
diff --git a/ajna-core/src/RewardsManager.sol b/ajna-core/src/RewardsManager.sol
index 421940f..4cdfefa 100644
--- a/ajna-core/src/RewardsManager.sol
+++ b/ajna-core/src/RewardsManager.sol
@@ -792,13 +792,15 @@ contract RewardsManager is IRewardsManager, ReentrancyGuard {
                 (, , , uint256 bucketDeposit, ) = IPool(pool_).bucketInfo(bucketIndex_);
 
                 uint256 burnFactor     = Maths.wmul(totalBurned_, bucketDeposit);
-                uint256 interestFactor = interestEarned_ == 0 ? 0 : Maths.wdiv(
-                    Maths.WAD - Maths.wdiv(prevBucketExchangeRate, curBucketExchangeRate),
-                    interestEarned_
-                );
 
                 // calculate rewards earned for updating bucket exchange rate 
-                rewards_ += Maths.wmul(UPDATE_CLAIM_REWARD, Maths.wmul(burnFactor, interestFactor));
+                rewards_ += interestEarned_ == 0 ? 0 : Maths.wmul(UPDATE_CLAIM_REWARD, Maths.wdiv(
+                    Maths.wmul(
+                        Maths.WAD - Maths.wdiv(prevBucketExchangeRate, curBucketExchangeRate),
+                        burnFactor
+                    ),
+                    interestEarned_
+                ));
             }
         }
```

```shell
forge test --match-test testClaimRewardsFuzzy -vvvv

For a Fuzzing input:
indexes: 3
mintAmount: 73528480588506366763626

Divide first and multiply
emit CalculateReward(: 334143554965844407584)

Multiply first and divide
emit CalculateReward(: 334143554965846586903)
```

## Tools Used

Foundry   

## Recommended Mitigation Steps

As modified above, multiply first and then divide.   



## Assessed type

Math