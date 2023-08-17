## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-22

# [Inaccurate estimation of validation rewards from function ExpectedRewardAVA in MiniPoolManager.sol](https://github.com/code-423n4/2022-12-gogopool-findings/issues/122) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L560
https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/MinipoolManager.sol#L676


# Vulnerability details

## Impact

The validation rewards can be inaccuartedly displayed to user and the slahsed amount can be wrong when slashing happens.

## Proof of Concept

note the function below:

```solidity
	/// @notice Given a duration and an AVAX amt, calculate how much AVAX should be earned via validation rewards
	/// @param duration The length of validation in seconds
	/// @param avaxAmt The amount of AVAX the node staked for their validation period
	/// @return The approximate rewards the node should recieve from Avalanche for beign a validator
	function getExpectedAVAXRewardsAmt(uint256 duration, uint256 avaxAmt) public view returns (uint256) {
		ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
		uint256 rate = dao.getExpectedAVAXRewardsRate();
		return (avaxAmt.mulWadDown(rate) * duration) / 365 days;
	}
```

As outlined in the comment section, the function is intended to  calculate how much AVAX should be earned via validation rewards

Besides display the reward, this function is also used in the function slash.

```solidity
/// @notice Slashes the GPP of the minipool with the given index
/// @dev Extracted this because of "stack too deep" errors.
/// @param index Index of the minipool
function slash(int256 index) private {
	address nodeID = getAddress(keccak256(abi.encodePacked("minipool.item", index, ".nodeID")));
	address owner = getAddress(keccak256(abi.encodePacked("minipool.item", index, ".owner")));
	uint256 duration = getUint(keccak256(abi.encodePacked("minipool.item", index, ".duration")));
	uint256 avaxLiquidStakerAmt = getUint(keccak256(abi.encodePacked("minipool.item", index, ".avaxLiquidStakerAmt")));
	uint256 expectedAVAXRewardsAmt = getExpectedAVAXRewardsAmt(duration, avaxLiquidStakerAmt);
	uint256 slashGGPAmt = calculateGGPSlashAmt(expectedAVAXRewardsAmt);
	setUint(keccak256(abi.encodePacked("minipool.item", index, ".ggpSlashAmt")), slashGGPAmt);

	emit GGPSlashed(nodeID, slashGGPAmt);

	Staking staking = Staking(getContractAddress("Staking"));
	staking.slashGGP(owner, slashGGPAmt);
}
```

note the code:

```solidity
uint256 expectedAVAXRewardsAmt = getExpectedAVAXRewardsAmt(duration, avaxLiquidStakerAmt);
uint256 slashGGPAmt = calculateGGPSlashAmt(expectedAVAXRewardsAmt);
```

the slashedGGPAmt is calculated based on the AVAX reward amount.

However, the estimation of the validation rewards is not accurate.

According to the doc:

https://docs.avax.network/nodes/build/set-up-an-avalanche-node-with-microsoft-azure

> Running a validator and staking with Avalanche provides extremely competitive rewards of between 9.69% and 11.54% depending on the length you stake for.

This implies that the staking length affect staking rewards, but this is kind of vague. What is the exact implementation of the reward calculation?

The implementation is linked below:

https://github.com/ava-labs/avalanchego/blob/master/vms/platformvm/reward/calculator.go#L40

```Golang
// Reward returns the amount of tokens to reward the staker with.
//
// RemainingSupply = SupplyCap - ExistingSupply
// PortionOfExistingSupply = StakedAmount / ExistingSupply
// PortionOfStakingDuration = StakingDuration / MaximumStakingDuration
// MintingRate = MinMintingRate + MaxSubMinMintingRate * PortionOfStakingDuration
// Reward = RemainingSupply * PortionOfExistingSupply * MintingRate * PortionOfStakingDuration
func (c *calculator) Calculate(stakedDuration time.Duration, stakedAmount, currentSupply uint64) uint64 {
	bigStakedDuration := new(big.Int).SetUint64(uint64(stakedDuration))
	bigStakedAmount := new(big.Int).SetUint64(stakedAmount)
	bigCurrentSupply := new(big.Int).SetUint64(currentSupply)

	adjustedConsumptionRateNumerator := new(big.Int).Mul(c.maxSubMinConsumptionRate, bigStakedDuration)
	adjustedMinConsumptionRateNumerator := new(big.Int).Mul(c.minConsumptionRate, c.mintingPeriod)
	adjustedConsumptionRateNumerator.Add(adjustedConsumptionRateNumerator, adjustedMinConsumptionRateNumerator)
	adjustedConsumptionRateDenominator := new(big.Int).Mul(c.mintingPeriod, consumptionRateDenominator)

	remainingSupply := c.supplyCap - currentSupply
	reward := new(big.Int).SetUint64(remainingSupply)
	reward.Mul(reward, adjustedConsumptionRateNumerator)
	reward.Mul(reward, bigStakedAmount)
	reward.Mul(reward, bigStakedDuration)
	reward.Div(reward, adjustedConsumptionRateDenominator)
	reward.Div(reward, bigCurrentSupply)
	reward.Div(reward, c.mintingPeriod)

	if !reward.IsUint64() {
		return remainingSupply
	}

	finalReward := reward.Uint64()
	if finalReward > remainingSupply {
		return remainingSupply
	}

	return finalReward
}
```

note the reward calculation formula:

```solidity
// Reward returns the amount of tokens to reward the staker with.
//
// RemainingSupply = SupplyCap - ExistingSupply
// PortionOfExistingSupply = StakedAmount / ExistingSupply
// PortionOfStakingDuration = StakingDuration / MaximumStakingDuration
// MintingRate = MinMintingRate + MaxSubMinMintingRate * PortionOfStakingDuration
// Reward = RemainingSupply * PortionOfExistingSupply * MintingRate * PortionOfStakingDuration
```

However, in the current ExpectedRewardAVA, the implementation is just:

AVAX reward rate * avax amount * duration / 365 days.

```solidity
ProtocolDAO dao = ProtocolDAO(getContractAddress("ProtocolDAO"));
		uint256 rate = dao.getExpectedAVAXRewardsRate();
		return (avaxAmt.mulWadDown(rate) * duration) / 365 days;
```

Clearly, the implementation of the avalanche side is more sophicated and accurate than the implemented ExpectedRewardAVA.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the project make the ExpectedRewardAVA implementation match the implement 

https://github.com/ava-labs/avalanchego/blob/master/vms/platformvm/reward/calculator.go#L40