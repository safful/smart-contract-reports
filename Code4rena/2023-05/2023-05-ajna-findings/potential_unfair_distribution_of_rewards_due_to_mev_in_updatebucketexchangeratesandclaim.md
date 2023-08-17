## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-04

# [Potential unfair distribution of Rewards due to MEV in updateBucketExchangeRatesAndClaim](https://github.com/code-423n4/2023-05-ajna-findings/issues/373) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-core/src/RewardsManager.sol#L310-L318


# Vulnerability details

## Impact

This vulnerability allows malicious actors to exploit the reward system by frontrunning transactions and unfairly claiming rewards, thereby disincentivizing honest users from updating the bucket exchange rates and contributing to the system.

## Proof of Concept

The `updateBucketExchangeRatesAndClaim` function is publicly callable and serves two main purposes:

1. Updates the exchange rate of a list of buckets.
2. If eligible, the caller can claim `5%` of the rewards accumulated to each bucket since the last burn event, if it hasn't already been updated.
https://github.com/code-423n4/2023-05-ajna/blob/fc70fb9d05b13aee2b44be2cb652478535a90edd/ajna-core/src/RewardsManager.sol#L310-L318
```solidity
function updateBucketExchangeRatesAndClaim(
        address pool_,
        uint256[] calldata indexes_
    ) external override returns (uint256 updateReward) {
        updateReward = _updateBucketExchangeRates(pool_, indexes_);

        // transfer rewards to sender
        _transferAjnaRewards(updateReward);
    }

```

So, to summarize it's primary purpose is to incentivize people who keep the state updated. However, given the nature of the function (first come, first serve) it becomes very prone to MEV.

Consider the following scenario:
1. Alice is hard-working, non-technical and constantly keeps track of when to update the buckets so she can claim a small reward. Unfortunately, she becomes notorious for getting most of the rewards from updating the bucket exchange rate.
2. A malicious actor spots Alice's recent gains and creates a bot to front run **any** transactions to RewardsManager's `_updateBucketExchangeRateAndCalculateRewards`submitted by Alice.
3. The day after that, Alice again see's theres a small reward to claim, attempts to claim it, but she gets front runned by whoever set the bot.
4. Since Alice is non-technical, she cannot ever beat the bot so she is left with a broken heart and no longer able to claim rewards.

I believe the system should be made fair to everybody that wants to contribute, hence this is a vulnerability that should be taken care of to ensure the fair distribution of awards to people who care about the protocol instead of .

## Tools Used

Manual Review


## Recommended Mitigation Steps

I see potentially two solutions here:
1. Introduce a randomized reward mechanism, such as a lottery system or a probabilistic reward distribution for people who contribute to updating buckets. This could reduce the predictability of rewards and hence the potential for MEV exploitation.
2. Consider limiting the reward claim process to users who have staked in the rewards manager because they are the individuals that are directly affected if the bucket is not updated, because if its not updated for 14 days they won't be getting rewards. Additionally, you can couple it with a rate-limitting mechanism by implementing a maximum claim per address per time period



## Assessed type

MEV