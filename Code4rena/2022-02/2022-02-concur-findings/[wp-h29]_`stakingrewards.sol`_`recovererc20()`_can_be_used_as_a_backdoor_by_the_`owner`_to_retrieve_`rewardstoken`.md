## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [[WP-H29] `StakingRewards.sol` `recoverERC20()` can be used as a backdoor by the `owner` to retrieve `rewardsToken`](https://github.com/code-423n4/2022-02-concur-findings/issues/210) 

# Lines of code

https://github.com/code-423n4/2022-02-concur/blob/72b5216bfeaa7c52983060ebfc56e72e0aa8e3b0/contracts/StakingRewards.sol#L166-L176


# Vulnerability details

https://github.com/code-423n4/2022-02-concur/blob/72b5216bfeaa7c52983060ebfc56e72e0aa8e3b0/contracts/StakingRewards.sol#L166-L176

```solidity
    function recoverERC20(address tokenAddress, uint256 tokenAmount)
        external
        onlyOwner
    {
        require(
            tokenAddress != address(stakingToken),
            "Cannot withdraw the staking token"
        );
        IERC20(tokenAddress).safeTransfer(owner(), tokenAmount);
        emit Recovered(tokenAddress, tokenAmount);
    }
```

### Impact

Users can lose all the rewards to the malicious/compromised `owner`.

### Recommendation

Change to:

```solidity
 function recoverERC20(
    address tokenAddress,
    address to,
    uint256 amount
) external onlyOwner {
    require(tokenAddress != address(stakingToken) && tokenAddress != address(rewardsToken), "20");

    IERC20(tokenAddress).safeTransfer(to, amount);
    emit Recovered(tokenAddress, to, amount);
}
```

