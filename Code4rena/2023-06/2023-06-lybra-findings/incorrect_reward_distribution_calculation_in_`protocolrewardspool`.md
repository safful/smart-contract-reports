## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-10

# [Incorrect Reward Distribution Calculation in `ProtocolRewardsPool`](https://github.com/code-423n4/2023-06-lybra-findings/issues/604) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L190-L218


# Vulnerability details

# Incorrect Reward Distribution Calculation in `ProtocolRewardsPool`



This report highlights a vulnerability in the `ProtocolRewardsPool` contract. The `getReward()` function, designed to distribute rewards to users, uses an incorrect calculation method that can result in incorrect reward distribution.

## Impact

In the `ProtocolRewardsPool` contract, a user can call the `getReward()` function to receive the rewards. The function first tries to pay the reward using `eUSD` token, and if enough amount of tokens was not available, it will use `peUSD`, and `stableToken` in the next steps. However, the protocol compares the number of shares with the amount of reward to send the reward.  If one share corresponds to a value greater than 1 `eUSD`, which is typically the case, users can be overpaid when claiming rewards. This can result in a significant discrepancy between the actual reward amount and the amount distributed.

## Proof of Concept

When a user invokes the `ProtocolRewardsPool.getReward()` function, the contract attempts to distribute the rewards using the `EUSD` token:

[(ProtocolRewardsPool.sol#L190-L218)](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L190-L218)

```solidity
function getReward() external updateReward(msg.sender) {
    uint reward = rewards[msg.sender];
    if (reward > 0) {
        rewards[msg.sender] = 0;
        IEUSD EUSD = IEUSD(configurator.getEUSDAddress());
        uint256 balance = EUSD.sharesOf(address(this));
        uint256 eUSDShare = balance >= reward ? reward : reward - balance;
        EUSD.transferShares(msg.sender, eUSDShare);
        if (reward > eUSDShare) {
            ERC20 peUSD = ERC20(configurator.peUSD());
            uint256 peUSDBalance = peUSD.balanceOf(address(this));
            if (peUSDBalance >= reward - eUSDShare) {
                peUSD.transfer(msg.sender, reward - eUSDShare);
                emit ClaimReward(
                    msg.sender,
                    EUSD.getMintedEUSDByShares(eUSDShare),
                    address(peUSD),
                    reward - eUSDShare,
                    block.timestamp
                );
            } else {
                if (peUSDBalance > 0) {
                    peUSD.transfer(msg.sender, peUSDBalance);
                }
                ERC20 token = ERC20(configurator.stableToken());
                uint256 tokenAmount = ((reward - eUSDShare - peUSDBalance) *
                    token.decimals()) / 1e18;
                token.transfer(msg.sender, tokenAmount);
                emit ClaimReward(
                    msg.sender,
                    EUSD.getMintedEUSDByShares(eUSDShare),
                    address(token),
                    reward - eUSDShare,
                    block.timestamp
                );
            }
        } else {
            emit ClaimReward(
                msg.sender,
                EUSD.getMintedEUSDByShares(eUSDShare),
                address(0),
                0,
                block.timestamp
            );
        }
    }
}
```

 To determine the available shares for rewarding users, the function calculates the shares of the `eUSD` token held by the contract and compares it with the total reward to be distributed.

Here is the code snippet illustrating this calculation:

```solidity
uint256 balance = EUSD.sharesOf(address(this));
uint256 eUSDShare = balance >= reward ? reward : reward - balance;
```

However, the comparison of shares with the reward in this manner is incorrect.

Let's consider an example to understand the problem. Suppose `rewards[msg.sender]` is equal to \$10 worth of `eUSD`, and the shares held by the contract are 9 shares. If each share corresponds to \$10 worth `eUSD`, the contract mistakenly assumes it does not have enough balance to cover the entire reward, because it has 9 shares; however, having 9 shares is equivalent to having \$90 worth of `eUSD`. Consequently, it first sends 9 shares, equivalent to \$90 worth of `eUSD`, and then sends \$1 worth `peUSD`. However, the sum of these sent values is \$91 worth of `eUSD`, while the user's actual reward is only \$10 worth `eUSD`.

This issue can lead to incorrect reward distribution, causing users to receive significantly more or less rewards than they should.



## Tools Used

Manual Review



## Recommended Mitigation Steps

To address this issue, it is recommended to replace the usage of `eUSDShare` with `EUSD.getMintedEUSDByShares(eUSDShare)` in the following lines:

-   [ProtocolRewardsPool.sol#L198](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L198)
-   [ProtocolRewardsPool.sol#L201-L202](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L201-L202)
-   [ProtocolRewardsPool.sol#L209](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L209)

This ensures that the correct amount of `eUSD` is transferred to the user while maintaining the accuracy of reward calculations.


## Assessed type

Math