## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- edited-by-warden
- selected for report

# [StakingRewards: recoverERC20() can be used as a backdoor by the owner to retrieve rewardsToken](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/49) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L213-L223


# Vulnerability details

## Impact
Similar to https://github.com/code-423n4/2022-02-concur-findings/issues/210
StakingRewards.recoverERC20 rightfully checks against the stakingToken being sweeped away.
However, there’s no check against the rewardsToken.
This is the case of an admin privilege, which allows the owner to sweep the rewards tokens, perhaps as a way to rug depositors.
```
    function recoverERC20(address tokenAddress, uint256 tokenAmount)
        external
        onlyOwner
    {
        require(
            tokenAddress != address(stakingToken),
            "Cannot withdraw the staking token"
        );
        ERC20(tokenAddress).safeTransfer(owner, tokenAmount);
        emit Recovered(tokenAddress, tokenAmount);
    }
```
## Proof of Concept
https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L213-L223
## Tools Used
None
## Recommended Mitigation Steps
Add an additional check
```
        require(
            tokenAddress != address(rewardsToken),
            "Cannot withdraw the rewards token"
        );
```