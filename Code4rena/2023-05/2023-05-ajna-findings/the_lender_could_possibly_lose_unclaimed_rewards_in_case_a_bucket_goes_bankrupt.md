## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor disputed
- H-06

# [The lender could possibly lose unclaimed rewards in case a bucket goes bankrupt](https://github.com/code-423n4/2023-05-ajna-findings/issues/329) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L192-L199


# Vulnerability details

## Impact
When the lender calls `PositionManager.memorializePositions` method the following happens:
1. Records bucket indexes along with its deposit times and lpBalances
2. Transfers LP ownership from the lender to PositionManager contract.

In point 1, it checks if there is a previous deposit and the bucket went bankrupt after prior memorialization, then it zero out the previous tracked LP. However, the lender could still have unclaimed rewards. In this case, the lender loses the rewards due to the lack of claiming rewards before zeroing out the previous tracked LP balance. If you check claim rewards functionality in RewardsManager, the bucket being not bankrupt is not a requirement. Please note that claiming rewards relies on the tracked LP balance in PositionManager. 


## Proof of Concept

- `PositionManager.memorializePositions` method
	- check for previous deposits and zero out the previous tracked LP if bucket is bankrupt
	```
		// check for previous deposits
		if (position.depositTime != 0) {
			// check that bucket didn't go bankrupt after prior memorialization
			if (_bucketBankruptAfterDeposit(pool, index, position.depositTime)) {
				// if bucket did go bankrupt, zero out the LP tracked by position manager
				position.lps = 0;
			}
		}

	```
	https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L192-L199



- In RewardsManager, check `claimRewards` and `_claimRewards`  method. there is no a check for bucket's bankruptcy.

	https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L114

	https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L561



## Tools Used
Manual analysis

## Recommended Mitigation Steps


On memorializePositions, check if the lender already claimed his/her rewards before zeroing out the previous tracked LP.



## Assessed type

Other