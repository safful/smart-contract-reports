## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- H-10

# [missing isEpochClaimed validation](https://github.com/code-423n4/2023-05-ajna-findings/issues/132) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L135-L198


# Vulnerability details

## Impact
User can claim rewards even when is already claimed

## Proof of Concept

The _claimRewards function is using to calculate and send the reward to the caller but this function is no validating if isEpochClaimed mapping is true due that in claimRewards function is validated, see the stament in the following lines:

```
file: ajna-core/src/RewardsManager.sol
function claimRewards(
        uint256 tokenId_,
        uint256 epochToClaim_ 
    ) external override {
        StakeInfo storage stakeInfo = stakes[tokenId_];

        if (msg.sender != stakeInfo.owner) revert NotOwnerOfDeposit(); 

        if (isEpochClaimed[tokenId_][epochToClaim_]) revert AlreadyClaimed(); // checking if the epoch was claimed;

        _claimRewards(
            stakeInfo,
            tokenId_,
            epochToClaim_,
            true,
            stakeInfo.ajnaPool
        );
    }
```
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L114-L125

Now the moveStakedLiquidity is calling _claimRewards too without validate isEpochClaimed mapping:

```
file: ajna-core/src/RewardsManager.sol
function moveStakedLiquidity(
        uint256 tokenId_,
        uint256[] memory fromBuckets_,
        uint256[] memory toBuckets_,
        uint256 expiry_
    ) external override nonReentrant {
        StakeInfo storage stakeInfo = stakes[tokenId_];

        if (msg.sender != stakeInfo.owner) revert NotOwnerOfDeposit(); 

        uint256 fromBucketLength = fromBuckets_.length;
        if (fromBucketLength != toBuckets_.length)
            revert MoveStakedLiquidityInvalid();

        address ajnaPool = stakeInfo.ajnaPool;
        uint256 curBurnEpoch = IPool(ajnaPool).currentBurnEpoch();

        // claim rewards before moving liquidity, if any
        _claimRewards(stakeInfo, tokenId_, curBurnEpoch, false, ajnaPool); // no checking is isEpochClaimed is true and revert
```
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/RewardsManager.sol#L135-L159

Also we can see in the _claimRewards function there is no validation is isEpochClaimed is true, this allow  a malicius user claimReward first and then move his liquidity to other bucket or the same bucket claiming the reward each time that he want. 

```
function _claimRewards(
        StakeInfo storage stakeInfo_,
        uint256 tokenId_,
        uint256 epochToClaim_,
        bool validateEpoch_,
        address ajnaPool_
    ) internal {
        // revert if higher epoch to claim than current burn epoch
        if (
            validateEpoch_ &&
            epochToClaim_ > IPool(ajnaPool_).currentBurnEpoch()
        ) revert EpochNotAvailable();

        // update bucket exchange rates and claim associated rewards
        uint256 rewardsEarned = _updateBucketExchangeRates(
            ajnaPool_,
            positionManager.getPositionIndexes(tokenId_)
        );

        rewardsEarned += _calculateAndClaimRewards(tokenId_, epochToClaim_);

        uint256[] memory burnEpochsClaimed = _getBurnEpochsClaimed(
            stakeInfo_.lastClaimedEpoch,
            epochToClaim_
        );

        emit ClaimRewards(
            msg.sender,
            ajnaPool_,
            tokenId_,
            burnEpochsClaimed,
            rewardsEarned
        );

        // update last interaction burn event
        stakeInfo_.lastClaimedEpoch = uint96(epochToClaim_);

        // transfer rewards to sender
        _transferAjnaRewards(rewardsEarned);
    }
```

## Tools Used
manual

## Recommended Mitigation Steps
check if the isEpochClaime is true and revert in the _claimReward function

```
if (isEpochClaimed[tokenId_][epochToClaim_]) revert AlreadyClaimed();
```


## Assessed type

Invalid Validation