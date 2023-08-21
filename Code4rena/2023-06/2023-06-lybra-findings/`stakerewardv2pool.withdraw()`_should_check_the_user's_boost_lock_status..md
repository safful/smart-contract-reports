## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-07

# [`stakerewardV2pool.withdraw()` should check the user's boost lock status.](https://github.com/code-423n4/2023-06-lybra-findings/issues/773) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/miner/stakerewardV2pool.sol#L93


# Vulnerability details

## Impact
Users can withdraw their staking token immediately after charging more rewards using boost.

## Proof of Concept
`withdraw()` should prevent withdrawals during the boost lock but there is no such logic.

The below steps show how users can charge more rewards without locking their funds.
1. Alice stakes her funds using [stake()](https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/miner/stakerewardV2pool.sol#L83).
2. She sets the longest lock duration to get the highest boost using [setLockStatus()](https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/miner/esLBRBoost.sol#L38).
3. After that, when she wants to withdraw her staking funds, she calls [withdraw()](https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/miner/stakerewardV2pool.sol#L93).
```solidity
    function withdraw(uint256 _amount) external updateReward(msg.sender) {
        require(_amount > 0, "amount = 0");
        balanceOf[msg.sender] -= _amount;
        totalSupply -= _amount;
        stakingToken.transfer(msg.sender, _amount);
        emit WithdrawToken(msg.sender, _amount, block.timestamp);
    }
```
4. Then the highest boost factor will be applied to her rewards in [earned()](https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/miner/stakerewardV2pool.sol#L106) and she can withdraw all of her staking funds and rewards immediately without checking any lock duration.
```solidity
    // Calculates and returns the earned rewards for a user
    function earned(address _account) public view returns (uint256) {
        return ((balanceOf[_account] * getBoost(_account) * (rewardPerToken() - userRewardPerTokenPaid[_account])) / 1e38) + rewards[_account];
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
`withdraw()` should check the boost lock like this.

```solidity
    function withdraw(uint256 _amount) external updateReward(msg.sender) {
        require(block.timestamp >= esLBRBoost.getUnlockTime(msg.sender), "Your lock-in period has not ended.");

        require(_amount > 0, "amount = 0");
        balanceOf[msg.sender] -= _amount;
        totalSupply -= _amount;
        stakingToken.transfer(msg.sender, _amount);
        emit WithdrawToken(msg.sender, _amount, block.timestamp);
    }
```


## Assessed type

Invalid Validation