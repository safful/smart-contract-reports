## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-10

# [Unsafe casting from `uint256` to `uint128` in RewardsManager](https://github.com/code-423n4/2023-05-ajna-findings/issues/227) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L179
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L180
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L236
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L241


# Vulnerability details

## Impact
Unsafe casting from `uint256` to `uint128` in RewardsManager, instances:
```diff
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

178:             BucketState storage toBucket = stakeInfo.snapshot[toIndex];
+179:             toBucket.lpsAtStakeTime  = uint128(positionManager.getLP(tokenId_, toIndex));
+180:             toBucket.rateAtStakeTime = uint128(IPool(ajnaPool).bucketExchangeRate(toIndex));
181: 

235:             // record the number of lps in bucket at the time of staking
+236:             bucketState.lpsAtStakeTime = uint128(positionManager.getLP(
237:                 tokenId_,
238:                 bucketId
239:             ));
240:             // record the bucket exchange rate at the time of staking
+241:             bucketState.rateAtStakeTime = uint128(IPool(ajnaPool).bucketExchangeRate(bucketId));
```
can cause an overflow which, in turn, can lead to unforeseen consequences such as:
*  The inability to calculate new rewards, as `nextExchangeRate > exchangeRate_` will always be true after the overflow.
* Reduced rewards because `toBucket.lpsAtStakeTime` will be reduced.
* Reduced rewards because `toBucket.rateAtStakeTime` will be reduced.
* In case `bucketState.rateAtStakeTime` overflows first but does not go beyond the limits in the new epoch, it will result in increased rewards being accrued.

## Proof of Concept
In `RewardsManager.stake()` and `RewardsManager.moveStakedLiquidity()`, the functions downcast `uint256` to `uint128` without checking whether it is bigger than `uint128` or not.

In `stake()` & `moveStakedLiquidity()` when `getLP >= type(uint128).max`:
```javascript
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

236:             bucketState.lpsAtStakeTime = uint128(positionManager.getLP(
237:                 tokenId_,
238:                 bucketId
239:             ));

```
Let's assume, that the staker had LP in the bucket equal `type(uint128).max`, and his stake balance LP was recorded as 0. As a result, the reward for the epoch at the moment of the stake will be accrued as

```javascript
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

504:                 interestEarned_ = Maths.wmul(nextExchangeRate - exchangeRate_, bucketLP_);

2023-05-ajna\ajna-core\src\libraries\internal\Maths.sol
11:     uint256 internal constant WAD = 1e18;
12: 
13:     function wmul(uint256 x, uint256 y) internal pure returns (uint256) {
14:         return (x * y + WAD / 2) / WAD;
15:     }
```
And will be equal to (0 + 0.5e18)/1e18=0, resulting in the user losing the reward.

In `stake() & moveStakedLiquidity()` when `bucketExchangeRate >= type(uint128).max`:
If an overflow occurs and `bucketExchangeRate` is reset to zero:
```javascript
File: 2023-05-ajna\ajna-core\src\RewardsManager.sol

180:             toBucket.rateAtStakeTime = uint128(IPool(ajnaPool).bucketExchangeRate(toIndex));

241:             bucketState.rateAtStakeTime = uint128(IPool(ajnaPool).bucketExchangeRate(bucketId));

```
Results in the reward being skipped for one 1 epoch, because:

```javascript

File: ajna-core\src\RewardsManager.sol

497:             uint256 nextExchangeRate = bucketExchangeRates[pool_][bucketIndex_][nextEventEpoch_];
498: 
499:             // calculate interest earned only if next exchange rate is higher than current exchange rate
500:             if (nextExchangeRate > exchangeRate_) {
501: 
502:                 // calculate the equivalent amount of quote tokens given the stakes lp balance,
503:                 // and the exchange rate at the next and current burn events
504:                 interestEarned_ = Maths.wmul(nextExchangeRate - exchangeRate_, bucketLP_);
505:             }
```
The current rate will be equal to 0 or greater than 0, but less than the previous rate.

Also, if the next epoch has a rate less than `type(uint128).max`, this will result in ```interestEarned_ = Maths.wmul(nextExchangeRate - exchangeRate_, bucketLP_);```, where `nextExchangeRate - exchangeRate_` will be in the range of `2^128 - 1 - {0, N^18}`. This can lead to an overflow error when `bucketLP_` is large `(1e45)`, because `(2^128-1) * 1e38`, which in turn can cause the transaction to fail.
```c
File: ajna-core\src\libraries\internal\Maths.sol

13:     function wmul(uint256 x, uint256 y) internal pure returns (uint256) {
14:         return (x * y + WAD / 2) / WAD;
15:     }
```
Yes, in the case of LP, the number of tokens approaching `2^128` is highly unlikely and does not pose a direct threat to the user, except for not receiving the reward. But as for the bucketExchangeRate, since it is calculated according to different formulas, it cannot be ruled out that such a case is more likely



**Tests for LP, which simply shows that overflow occurs:**
```diff

diff --git a/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol b/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
index 4100e9f..58c3d8c 100644
--- a/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
+++ b/ajna-core/tests/forge/unit/Rewards/RewardsManager.t.sol
@@ -8,7 +8,7 @@ import 'src/interfaces/rewards/IRewardsManager.sol';

 import { ERC20Pool }           from 'src/ERC20Pool.sol';
 import { RewardsHelperContract }   from './RewardsDSTestPlus.sol';
-
+import '@std/console2.sol';
 contract RewardsManagerTest is RewardsHelperContract {

     address internal _borrower;
@@ -127,6 +127,33 @@ contract RewardsManagerTest is RewardsHelperContract {
         });
     }

+    function test_Issue() external {
+        // configure NFT position one
+        uint256[] memory depositIndexes = new uint256[](1);
+        depositIndexes[0] = 9;
+        uint256 mintAmount = uint256(type(uint128).max) + 1;
+        uint256 tokenIdOne = _mintAndMemorializePositionNFT({
+            indexes:    depositIndexes,
+            minter:     _minterOne,
+            mintAmount: mintAmount,
+            pool:       address(_pool)
+        });
+        uint256 lpBalance;
+        (lpBalance, ) =_pool.lenderInfo(depositIndexes[0], address(_positionManager));
+        console2.log("_pool.lenderInfo for _positionManager before stake", lpBalance);
+
+        // minterOne deposits their NFT into the rewards contract
+        _stakeToken({
+            pool:    address(_pool),
+            owner:   _minterOne,
+            tokenId: tokenIdOne
+        });
+        uint256 lpsAtStakeTime;
+        uint256 rateAtStakeTime;
+        (lpsAtStakeTime, rateAtStakeTime) = _rewardsManager.getBucketStateStakeInfo(tokenIdOne, depositIndexes[0]);
+        console2.log("getBucketStateStakeInfo.lpsAtStakeTime after stake", lpsAtStakeTime);
+    }
+    
```
## Tools Used
* Manual review
* Foundry

## Recommended Mitigation Steps
* use OpenZeppelin’s SafeCast library when casting from uint256 to uint128.
* don't make casting from uint256 to uint128



## Assessed type

Under/Overflow