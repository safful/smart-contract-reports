## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-06

# [withdrawProtocolFees() Possible malicious or accidental withdrawal of all rewards](https://github.com/code-423n4/2023-05-maia-findings/issues/731) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/boost-aggregator/BoostAggregator.sol#L159-L161


# Vulnerability details

## Impact
`claimReward()` will take all rewards if the `amountRequested` passed in is 0, which may result in user's rewards lost

## Proof of Concept
In `BoostAggregator.withdrawProtocolFees()`, the owner can take the protocol rewards `protocolRewards`
The code is as follows:
```solidity
    function withdrawProtocolFees(address to) external onlyOwner {
        uniswapV3Staker.claimReward(to, protocolRewards);
@>      delete protocolRewards;
    }
```

From the above code we can see that `uniswapV3Staker` is called to fetch and then clear `protocolRewards`

Let's look at the implementation of `uniswapV3Staker.claimReward()`.

```solidity
contract UniswapV3Staker is IUniswapV3Staker, Multicallable {
....
    function claimReward(address to, uint256 amountRequested) external returns (uint256 reward) {
        reward = rewards[msg.sender];
@>      if (amountRequested != 0 && amountRequested < reward) {
            reward = amountRequested;
            rewards[msg.sender] -= reward;
        } else {
            rewards[msg.sender] = 0;
        }

        if (reward > 0) hermes.safeTransfer(to, reward);

        emit RewardClaimed(to, reward);
    }
```

The current implementation is that if the `amountRequested==0` passed in means that all `rewards[msg.sender]` of this `msg.sender` are taken



This leads to problems.
1. If a malicious `owner` calls `withdrawProtocolFees()` twice in a row, it will definitely take all the `rewards` of the `BoostAggregator`.
2. probably didn't realize that `withdrawProtocolFees()` was called when `protocolRewards==0`

As a result, the rewards that belong to users in `BoostAggregator` are lost



## Tools Used

## Recommended Mitigation Steps

Modify `claimReward()` to remove `amountRequested != 0`

```solidity
contract UniswapV3Staker is IUniswapV3Staker, Multicallable {
....
    function claimReward(address to, uint256 amountRequested) external returns (uint256 reward) {
        reward = rewards[msg.sender];
-       if (amountRequested != 0 && amountRequested < reward) {        
+       if (amountRequested < reward) {
            reward = amountRequested;
            rewards[msg.sender] -= reward;
        } else {
            rewards[msg.sender] = 0;
        }

        if (reward > 0) hermes.safeTransfer(to, reward);

        emit RewardClaimed(to, reward);
    }
```


## Assessed type

Context