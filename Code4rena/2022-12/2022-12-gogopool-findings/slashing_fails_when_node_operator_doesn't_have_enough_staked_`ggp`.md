## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-13

# [slashing fails when node operator doesn't have enough staked `GGP`](https://github.com/code-423n4/2022-12-gogopool-findings/issues/494) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/Staking.sol#L379-L383


# Vulnerability details

## Description
When creating a minipool the node operator is required to put up a collateral in `GGP`, the protocol token. The amount of `GGP` collateral needed is currently calculated to be 10% of the `AVAX` staked. This is calculated using the price of `GGP - AVAX`.

If the node operator doesn't have high enough availability and doesn't get any rewards the protocol will slash their `GGP` collateral to reward liquid stakers. This is also calculated using the price of `GGP - AVAX`:

```javascript
File: MinipoolManager.sol

547:	function calculateGGPSlashAmt(uint256 avaxRewardAmt) public view returns (uint256) {
548:		Oracle oracle = Oracle(getContractAddress("Oracle"));
549:		(uint256 ggpPriceInAvax, ) = oracle.getGGPPriceInAVAX(); // price might change or be manipulated
550:		return avaxRewardAmt.divWadDown(ggpPriceInAvax);
551:	}

...

670:	function slash(int256 index) private {

...

673:		uint256 duration = getUint(keccak256(abi.encodePacked("minipool.item", index, ".duration")));
674:		uint256 avaxLiquidStakerAmt = getUint(keccak256(abi.encodePacked("minipool.item", index, ".avaxLiquidStakerAmt")));
675:		uint256 expectedAVAXRewardsAmt = getExpectedAVAXRewardsAmt(duration, avaxLiquidStakerAmt);
676:		uint256 slashGGPAmt = calculateGGPSlashAmt(expectedAVAXRewardsAmt);

...

681:		Staking staking = Staking(getContractAddress("Staking"));
682:		staking.slashGGP(owner, slashGGPAmt);
683:	}
```

This is then subtracted from their staked amount:

```javascript
File: Staking.sol

94: 	function decreaseGGPStake(address stakerAddr, uint256 amount) internal {
95: 		int256 stakerIndex = requireValidStaker(stakerAddr);
96: 		subUint(keccak256(abi.encodePacked("staker.item", stakerIndex, ".ggpStaked")), amount); // can fail due to underflow
97: 	}

...

379:	function slashGGP(address stakerAddr, uint256 ggpAmt) public onlySpecificRegisteredContract("MinipoolManager", msg.sender) {
380:		Vault vault = Vault(getContractAddress("Vault"));
381:		decreaseGGPStake(stakerAddr, ggpAmt);
382:		vault.transferToken("ProtocolDAO", ggp, ggpAmt);
383:	}
```

The issue is that the current staked amount is never checked so the `subUint` can fail due to underflow if the price has changed since the minipool was created/recreated.

## Impact
If a node operator doesn't have enough collateral, possibly caused by price changes in `GGP` during slashing they evade slashing all together.

Its even possible for the node operator to foresee this and manipulate the price of `GGP` just prior to the period ending if they know that they are going to be slashed.

## Proof of Concept
PoC test in `MinipoolManager.t.sol`:

```javascript
	function testRecordStakingEndWithSlashNotEnoughStake() public {
		uint256 duration = 365 days;
		uint256 depositAmt = 1000 ether;
		uint256 avaxAssignmentRequest = 1000 ether;
		uint256 validationAmt = depositAmt + avaxAssignmentRequest;
		uint128 ggpStakeAmt = 100 ether; // just enough

		vm.startPrank(nodeOp);
		ggp.approve(address(staking), MAX_AMT);
		staking.stakeGGP(ggpStakeAmt);
		MinipoolManager.Minipool memory mp1 = createMinipool(depositAmt, avaxAssignmentRequest, duration);
		vm.stopPrank();

		address liqStaker1 = getActorWithTokens("liqStaker1", MAX_AMT, MAX_AMT);
		vm.prank(liqStaker1);
		ggAVAX.depositAVAX{value: MAX_AMT}();

		vm.prank(address(rialto));
		minipoolMgr.claimAndInitiateStaking(mp1.nodeID);

		bytes32 txID = keccak256("txid");
		vm.prank(address(rialto));
		minipoolMgr.recordStakingStart(mp1.nodeID, txID, block.timestamp);

		skip(2 weeks);
		
		vm.prank(address(rialto)); // price changes just a bit
		oracle.setGGPPriceInAVAX(0.999 ether, block.timestamp);

		vm.prank(address(rialto));
		vm.expectRevert(); // staking cannot end because of underflow
		minipoolMgr.recordStakingEnd{value: validationAmt}(mp1.nodeID, block.timestamp, 0 ether);
	}
```

The only thing the protocol can do now is to call `recordStakingError` for the minipool, since no other state changes are allowed. This will return the staked funds but it will not slash the `GGP` amount for the node operator. Hence the node operator has evaded the slashing.

## Tools Used
vs code, forge

## Recommended Mitigation Steps
If the amount to be slashed is greater than what the node operator has staked, slash all their stake.