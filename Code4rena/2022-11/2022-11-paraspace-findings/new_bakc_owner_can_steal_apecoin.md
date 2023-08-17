## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-06

# [New BAKC Owner Can Steal ApeCoin](https://github.com/code-423n4/2022-11-paraspace-findings/issues/284) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/ApeStakingLogic.sol#L38


# Vulnerability details

## Background

This section provides a context of the pre-requisite concepts that a reader needs to fully understand the issue.

#### Split Pair Edge Case In Paired Pool

Assume that Jimmy is the owner of BAYC #8888 and BAKC #9999 NFTs initially. He participated in the Paired Pool and staked + accrued a total of 100 ApeCoin (APE) at this point, as shown in the diagram below.

Jimmy then sold his BAKC #9999 NFT to Ben. When this happens, both parties (Jimmy and Ben) could close out their staking position. Since Ben owns BAKC #9999 now, he can close out Jimmy's position anytime and claim all the accrued APE rewards (2 APE below). While Jimmy will obtain the 98 APE that he staked initially.

The following image is taken from https://youtu.be/_LO-1f9nyjs?t=640

![](https://user-images.githubusercontent.com/102820284/206686601-3c34a2a1-6b80-420d-8ed8-2f86ab6ca103.png)

The `ApeCoinStaking._withdrawPairNft` taken from the official $APE Staking Contract shows that the implementation allows both the BAYC/MAYC owners and BAKC owners to close out the staking position. Refer to Line 976 below.

When the staking position is closed by the BAKC owners, the entire staking amount must be withdrawn. A partial amount is not allowed per Line 981 below. In Line 984, all the accrued APE rewards will be sent to the BAKC owners. In Line 989, all the staked APEs will be withdrawn (unstake) and sent directly to the wallet of the BAYC owners.

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/dependencies/yoga-labs/ApeCoinStaking.sol#L966

```solidity
File: ApeCoinStaking.sol
966:     function _withdrawPairNft(uint256 mainTypePoolId, PairNftWithAmount[] calldata _nfts) private {
967:         for(uint256 i; i < _nfts.length; ++i) {
968:             uint256 mainTokenId = _nfts[i].mainTokenId;
969:             uint256 bakcTokenId = _nfts[i].bakcTokenId;
970:             uint256 amount = _nfts[i].amount;
971:             address mainTokenOwner = nftContracts[mainTypePoolId].ownerOf(mainTokenId);
972:             address bakcOwner = nftContracts[BAKC_POOL_ID].ownerOf(bakcTokenId);
973:             PairingStatus memory mainToSecond = mainToBakc[mainTypePoolId][mainTokenId];
974:             PairingStatus memory secondToMain = bakcToMain[bakcTokenId][mainTypePoolId];
975: 
976:             require(mainTokenOwner == msg.sender || bakcOwner == msg.sender, "At least one token in pair must be owned by caller");
977:             require(mainToSecond.tokenId == bakcTokenId && mainToSecond.isPaired
978:                 && secondToMain.tokenId == mainTokenId && secondToMain.isPaired, "The provided Token IDs are not paired");
979: 
980:             Position storage position = nftPosition[BAKC_POOL_ID][bakcTokenId];
981:             require(mainTokenOwner == bakcOwner || amount == position.stakedAmount, "Split pair can't partially withdraw");
982: 
983:             if (amount == position.stakedAmount) {
984:                 uint256 rewardsToBeClaimed = _claim(BAKC_POOL_ID, position, bakcOwner);
985:                 mainToBakc[mainTypePoolId][mainTokenId] = PairingStatus(0, false);
986:                 bakcToMain[bakcTokenId][mainTypePoolId] = PairingStatus(0, false);
987:                 emit ClaimRewardsPairNft(msg.sender, rewardsToBeClaimed, mainTypePoolId, mainTokenId, bakcTokenId);
988:             }
989:             _withdraw(BAKC_POOL_ID, position, amount, mainTokenOwner);
990:             emit WithdrawPairNft(msg.sender, amount, mainTypePoolId, mainTokenId, bakcTokenId);
991:         }
992:     }
```

The following shows that the `ApeCoinStaking._withdraw` used in the above function will send the unstaked APEs directly to the wallet of BAYC owners. Refer to Line 946 below.

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/dependencies/yoga-labs/ApeCoinStaking.sol#L937

```solidity
File: ApeCoinStaking.sol
937:     function _withdraw(uint256 _poolId, Position storage _position, uint256 _amount, address _recipient) private {
938:         require(_amount <= _position.stakedAmount, "Can't withdraw more than staked amount");
939: 
940:         Pool storage pool = pools[_poolId];
941: 
942:         _position.stakedAmount -= _amount;
943:         pool.stakedAmount -= _amount;
944:         _position.rewardsDebt -= (_amount * pool.accumulatedRewardsPerShare).toInt256();
945: 
946:         apeCoin.safeTransfer(_recipient, _amount);
947:     }
```

#### Who is the owner of BAYC/MAYC in ParaSpace?

The BAYC is held by the `NTokenBAYC` reserve pool, while the MAYC is held by `NTokenMAYC` reserve pool. This causes an issue because, as mentioned in the previous section, all the unstaked APE will be sent directly to the wallet of the BAYC/MAYC owners. This will be used as part of the attack path later on.

#### Does BAKC need to be staked or stored within ParaSpace?

No. For the purpose of APE staking via ParaSpace , BAKC NFT need not be held in the ParaSpace contract for staking, but Bored Apes and Mutant Apes must be collateralized in the ParaSpace protocol. Refer to [here](https://docs.para.space/para-space/introduction-to-paraspace/borrow-and-stake-apecoin-with-bored-ape-yacht-club-nfts#deposit-and-stake-unstake-and-withdraw). As such, users are free to sell off their BAKC anytime to anyone since it is not being locked up.

## Proof of Concept

Using back the same example in the previous section. Assume the following:

- Jimmy is the victim, and Ben is the attacker.
- Jimmy is the owner of BAYC #8888 and BAKC #9999 NFTs initially. He participated in the Paired Pool and staked + accrued a total of 100 ApeCoin (APE).
- Jimmy sold his BAKC #9999 NFT to Ben. There are many ways Ben can obtain the BAKC #9999 NFT. Ben could obtain it via the public marketplace (e.g. Opensea) if Jimmy listed it OR Ben could offer an attractive price to Jimmy to purchase it privately.
- Ben also participates in the APE staking in ParaSpace via his BAYC #0002 and BAKC #0002 NFTs.

Ben will close out Jimmy's position by calling the `ApeCoinStaking.withdrawBAKC` function of the official $APE Staking Contract to withdraw all the staked APE + accrued APE rewards. As a result, the 98 APE that Jimmy staked initially will be sent directly to the owner of the BAYC #8888 owner. In this case, the BAYC #8888 owner is ParaSpace's `NTokenBAYC` reserve pool.

At this point, it is important to note that Jimmy's 98 APE is stuck in ParaSpace's `NTokenBAYC` reserve pool. If anyone can siphon out Jimmy's 98 APE that is stuck in the contract, that person will be able to get free APE.

There exist a way to siphon out all the APE coin that resides in ParaSpace's `NTokenBAYC` reserve pool. Anyone who participates in APE staking via ParaSpace with a BAYC, which also means that any user staked in the Paired Pool, can trigger the `ApeStakingLogic.withdrawBAKC` function below by calling the `PoolApeStaking.withdrawBAKC` function.

Notice that in Line 53 below, it will compute the total balance of APE held by ParaSpace's `NTokenBAYC` reserve pool contract. In Line 55, it will send all the APE in the pool contract to the recipient. This effectively performs a sweeping of APE stored in the pool contract.

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/ApeStakingLogic.sol#L38

```solidity
File: ApeStakingLogic.sol
38:     function withdrawBAKC(
39:         ApeCoinStaking _apeCoinStaking,
40:         uint256 poolId,
41:         ApeCoinStaking.PairNftWithAmount[] memory _nftPairs,
42:         address _apeRecipient
43:     ) external {
44:         ApeCoinStaking.PairNftWithAmount[]
45:             memory _otherPairs = new ApeCoinStaking.PairNftWithAmount[](0);
46: 
47:         if (poolId == BAYC_POOL_ID) {
48:             _apeCoinStaking.withdrawBAKC(_nftPairs, _otherPairs);
49:         } else {
50:             _apeCoinStaking.withdrawBAKC(_otherPairs, _nftPairs);
51:         }
52: 
53:         uint256 balance = _apeCoinStaking.apeCoin().balanceOf(address(this));
54: 
55:         _apeCoinStaking.apeCoin().safeTransfer(_apeRecipient, balance);
56:     }
```

Thus, after Ben closes out Jimmy's position by calling the `ApeCoinStaking.withdrawBAKC` function and causes the 98 APE to be sent to ParaSpace's `NTokenBAYC` reserve pool contract, Ben immediately calls the `PoolApeStaking.withdrawBAKC` function against his own BAYC #0002 and BAKC #0002 NFTs. This will result in all of Jimmy's 98 APE being swept to his wallet. Bob effectively steals Jimmy's 98 APE.

## Impact

ApeCoin of ParaSpace users can be stolen.

## Recommended Mitigation Steps

Consider the potential side effects of the split pair edge case in the pair pool when implementing the APE staking feature in ParaSpace.

The official APE staking contract has been implemented recently, and only a few protocols have integrated with it. Thus, the edge cases are not widely understood and are prone to errors.

To mitigate this issue, instead of returning BAKC NFT to the users after staking, consider locking up BAKC NFT in the ParaSpace contract as part of the user's collateral. In this case, the user will not be able to sell off their BAKC, and the "Split Pair Edge Case In Paired Pool" scenario will not happen.