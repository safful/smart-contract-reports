## Tags

- 2 (Med Risk)
- selected for report
- edited-by-warden
- MR-M-08

# [`amountAvailableForStaking()` not fully utilized with `compoundedAvaxNodeOpAmt` easily forfeited](https://github.com/code-423n4/2023-02-gogopool-mitigation-contest-findings/issues/52) 

# Lines of code

https://github.com/multisig-labs/gogopool/blob/3b5ab1d6505ef9be6197c4056acd38d6bed4aff6/contracts/contract/tokens/TokenggAVAX.sol#L134-L146
https://github.com/multisig-labs/gogopool/blob/3b5ab1d6505ef9be6197c4056acd38d6bed4aff6/contracts/contract/MinipoolManager.sol#L497-L505


# Vulnerability details

## Impact
The mitigated step is implemented at the expense of economic loss to both the node operators and the liquid stakers if `compoundedAvaxNodeOpAmt <= ggAVAX.amountAvailableForStaking()`.

## Proof of Concept
Here is a typical scenario:

1. The protocol now assumes that a 1:1 nodeOp:liqStaker funds ratio is guaranteed to be met because of the atomic transaction that has also been implemented.
2. This is deemed an edge case that will only be optimally utilized if `compoundedAvaxAmt == ggAVAX.amountAvailableForStaking()`.
3. The atomic transaction is going to fail if `compoundedAvaxAmt > ggAVAX.amountAvailableForStaking()` after all due to situations like liquid stakers have been actively calling [`withdrawAVAX()`](https://github.com/multisig-labs/gogopool/blob/3b5ab1d6505ef9be6197c4056acd38d6bed4aff6/contracts/contract/tokens/TokenggAVAX.sol#L196-L205).

Under normal circumstances, [`ggAVAX.amountAvailableForStaking()`](https://github.com/multisig-labs/gogopool/blob/3b5ab1d6505ef9be6197c4056acd38d6bed4aff6/contracts/contract/tokens/TokenggAVAX.sol#L134-L146) is going to be adequate enough to cater for `compoundedAvaxNodeOpAmt`. This should not be easily forfeited without first checking whether or not `ggAVAX.amountAvailableForStaking()` is greater than `compoundedAvaxNodeOpAmt`.

## Tools Used
Manual inspection

## Recommended Mitigation Steps
Consider implementing the following check in `recreateMinipool()` to get the best out of it:

[File: MinipoolManager.sol#L486-L517](https://github.com/multisig-labs/gogopool/blob/3b5ab1d6505ef9be6197c4056acd38d6bed4aff6/contracts/contract/MinipoolManager.sol#L486-L517)

```diff
	function recreateMinipool(address nodeID) internal whenNotPaused {
		int256 minipoolIndex = onlyValidMultisig(nodeID);
		Minipool memory mp = getMinipool(minipoolIndex);
		MinipoolStatus currentStatus = MinipoolStatus(mp.status);

		if (currentStatus != MinipoolStatus.Withdrawable) {
			revert InvalidStateTransition();
		}

+                uint256 compoundedAvaxAmt;
+                uint256 rewardAmt;
+		if (mp.avaxNodeOpAmt + mp.avaxNodeOpRewardAmt <= ggAVAX.amountAvailableForStaking()) {
+                        compoundedAvaxAmt = mp.avaxNodeOpAmt + mp.avaxNodeOpRewardAmt;
+                        rewardAmt = mp.avaxNodeOpRewardAmt;
+                } else {
+                        compoundedAvaxAmt = mp.avaxNodeOpAmt + mp.avaxLiquidStakerRewardAmt;
+                        rewardAmt = mp.avaxLiquidStakerRewardAmt;
+                }

		// Compound the avax plus rewards
		// NOTE Assumes a 1:1 nodeOp:liqStaker funds ratio
-		uint256 compoundedAvaxAmt = mp.avaxNodeOpAmt + mp.avaxLiquidStakerRewardAmt;
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxNodeOpAmt")), compoundedAvaxAmt);
		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".avaxLiquidStakerAmt")), compoundedAvaxAmt);

		Staking staking = Staking(getContractAddress("Staking"));
		// Only increase AVAX stake by rewards amount we are compounding
		// since AVAX stake is only decreased by withdrawMinipool()
-		staking.increaseAVAXStake(mp.owner, mp.avaxLiquidStakerRewardAmt);
+		staking.increaseAVAXStake(mp.owner, rewardAmt);
		staking.increaseAVAXAssigned(mp.owner, compoundedAvaxAmt);

		if (staking.getRewardsStartTime(mp.owner) == 0) {
			// Edge case where calculateAndDistributeRewards has reset their rewards time even though they are still cycling
			// So we re-set it here to their initial start time for this minipool
			staking.setRewardsStartTime(mp.owner, mp.initialStartTime);
		}

		ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
		uint256 ratio = staking.getCollateralizationRatio(mp.owner);
		if (ratio < dao.getMinCollateralizationRatio()) {
			revert InsufficientGGPCollateralization();
		}

		resetMinipoolData(minipoolIndex);

		setUint(keccak256(abi.encodePacked("minipool.item", minipoolIndex, ".status")), uint256(MinipoolStatus.Prelaunch));

		emit MinipoolStatusChanged(nodeID, MinipoolStatus.Prelaunch);
	}
```