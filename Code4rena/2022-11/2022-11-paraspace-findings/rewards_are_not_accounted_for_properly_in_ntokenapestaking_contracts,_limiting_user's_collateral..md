## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-19

# [Rewards are not accounted for properly in NTokenApeStaking contracts, limiting user's collateral.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/481) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/ApeStakingLogic.sol#L253


# Vulnerability details

## Description

ApeStakingLogic.sol implements the logic for staking ape coins through the NTokenApeStaking NFT.

`getTokenIdStakingAmount()` is an important function which returns the entire stake amount mapping for a specific BAYC / MAYC NFT.

```
function getTokenIdStakingAmount(
    uint256 poolId,
    ApeCoinStaking _apeCoinStaking,
    uint256 tokenId
) public view returns (uint256) {
    (uint256 apeStakedAmount, ) = _apeCoinStaking.nftPosition(
        poolId,
        tokenId
    );
    uint256 apeReward = _apeCoinStaking.pendingRewards(
        poolId,
        address(this),
        tokenId
    );
    (uint256 bakcTokenId, bool isPaired) = _apeCoinStaking.mainToBakc(
        poolId,
        tokenId
    );
    if (isPaired) {
        (uint256 bakcStakedAmount, ) = _apeCoinStaking.nftPosition(
            BAKC_POOL_ID,
            bakcTokenId
        );
        apeStakedAmount += bakcStakedAmount;
    }
    return apeStakedAmount + apeReward;
}
```

We can see that the total returned amount is the staked amount through the direct NFT, plus rewards for the direct NFT, plus the staked amount of the BAKC token paired to the direct NFT. However, the calculation does not include the pendingRewards for the BAKC staked amount, which accrues over time as well.

As a result, getTokenIdStakingAmount() returns a value lower than the correct user balance. This function is used in PTokenSApe.sol's balanceOf function, as this type of PToken is supposed to reflect the user's current balance in ape staking. 

When user unstakes their ape tokens through executeUnstakePositionAndRepay, they will receive their fair share of rewards.It will call ApeCoinStaking's \_withdrawPairNft which will claim rewards also for BAKC tokens. However, because balanceOf() shows a lower value, the rewards not count as collateral for user's debt, which is a major issue for lending platforms.

## Impact

Rewards are not accounted for properly in NTokenApeStaking contracts, limiting user's collateral.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Balance calculation should include pendingRewards from BAKC tokens if they exist.