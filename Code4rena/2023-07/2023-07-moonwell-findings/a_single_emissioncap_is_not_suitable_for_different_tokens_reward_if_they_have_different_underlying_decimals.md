## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-17

# [a single emissionCap is not suitable for different tokens reward if they have different underlying decimals](https://github.com/code-423n4/2023-07-moonwell-findings/issues/19) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L65-L68


# Vulnerability details

## Impact
a single emissionCap is not suitable for different tokens reward if they have different underlying decimals

At MultiRewardDistributor, different rewards token can be set up for each mToken, which is the assetToken or debtToken of the lending protocol. When a reward token is set up, it's emission speed cannot be bigger than the emissionCap (100 * 1e18 as suggested in the code comment). However since multiple reward tokens may be set up, and they may have different decimal places, for example USDT has a decimal of 6 and GUSD even has a decimal of 2 they are virtually not bounded by such emissionCap. 

On the contract, tokens with more than 18 decimal would not be practically emitted. This one-size-fit-all emissionCap might create obstacles for the protocol to effectively manage each rewardTokens.

```solidity
    /// @notice The emission cap dictates an upper limit for reward speed emission speed configs
    /// @dev By default, is set to 100 1e18 token emissions / sec to avoid unbounded
    ///  computation/multiplication overflows
    uint256 public emissionCap;
```

```solidity
    function _updateBorrowSpeed(
        MToken _mToken,
        address _emissionToken,
        uint256 _newBorrowSpeed
    ) external onlyEmissionConfigOwnerOrAdmin(_mToken, _emissionToken) {
        MarketEmissionConfig
            storage emissionConfig = fetchConfigByEmissionToken(
                _mToken,
                _emissionToken
            );

        uint256 currentBorrowSpeed = emissionConfig
            .config
            .borrowEmissionsPerSec;

        require(
            _newBorrowSpeed != currentBorrowSpeed,
            "Can't set new borrow emissions to be equal to current!"
        );
        require(
            _newBorrowSpeed < emissionCap,
            "Cannot set a borrow reward speed higher than the emission cap!"
        );
```
https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L703-L725


## Proof of Concept


## Tools Used

## Recommended Mitigation Steps
Consider creating emissionCap for each rewardTokens so they can be adapted based on their own decimals


## Assessed type

Context