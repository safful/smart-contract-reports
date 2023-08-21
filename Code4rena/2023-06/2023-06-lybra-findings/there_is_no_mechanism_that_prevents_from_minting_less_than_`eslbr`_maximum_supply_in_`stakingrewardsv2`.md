## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-09

# [There is no mechanism that prevents from minting less than `esLBR` maximum supply in `StakingRewardsV2`](https://github.com/code-423n4/2023-06-lybra-findings/issues/647) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L111-L118
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L106-L108
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/esLBR.sol#L30-L32
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/esLBR.sol#L20
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L73-L77


# Vulnerability details

Note: I'm assuming that `esLBR` is distributed as a reward in `StakingRewardsV2` - it's not clear from the docs, but `rewardsToken` is of type `IesLBR` and in order to calculate boost for rewards `esLBRBoost` contract is used, so I think that it's a reasonable assumpiton.


`esLBR` token has a total supply of `100 000 000` and this is enforced in the `esLBR` contract:
```solidity
function mint(address user, uint256 amount) external returns (bool) {
        require(configurator.tokenMiner(msg.sender), "not authorized");
        require(totalSupply() + amount <= maxSupply, "exceeding the maximum supply quantity.");
```
However, `StakingRewardsV2` contract which is approved to mint new `esLBR` tokens doesn't enforce that new tokens can always be minted.

Either due to admin mistake (it's possible to call `StakingRewardsV2::notifyRewardAmount` with arbitrarily high `_amount`, which is not validated; it's also possible to set `duration` to an arbitrarily low value, so `rewardRatio` may be very high), or by normal protocol functioning, `100 000 000` of `esLBR` may be finally minted. 

If that happens, no user will be able to claim his reward via `getReward`, since `mint` will revert. It also won't be possible to stake `esLBR` tokens in `ProtocolRewardsPool` and call any functions that use  `esLBR.mint` underneath.

## Impact
Lack of possibility to stake `esLBR` is impacting important functionality of the protocol, while no possibility to withdraw earned rewards is a loss of assets for users.

From Code4Rena docs:
```
3 — High: Assets can be stolen/lost/compromised directly (or indirectly if there is a valid attack path that does not have hand-wavy hypotheticals).
```
Here assets definitely can be lost. Also, while it could happen because of a misconfiguration by the admin, it can also happen naturally, since `esLBR` is inflationary and there is no mechanism that enforces the supply being far enough from the max supply - the only thing that could be done to prevent it, is that admin would have to calculate the current supply, analyse the number of stakers, control staking boosts, and set reward ratio accordingly, which is hard to do and error prone.
Since assets can be lost and there aren't needed any external requirements here (and it doesn't have hand-wavy hypotheticals, in my opinion), I'm submitting this finding as High.

## Proof of Concept
Number of reward tokens that users get is calculated here:
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L106-L108

Users can get their rewards by calling `getReward`:
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L111-L118

There is no mechanism preventing too high `rewardRatio` when it's set:
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L132-L145

`mint` will fail on too high supply:
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/esLBR.sol#L30-L36

Users won't be able to claim acquired rewards, which is a loss of assets for them.
## Tools Used
VS Code

## Recommended Mitigation Steps
Do one of the following:
- introduce some mechanism that will enforce that `esLBR` max supply will never be achieved (something similar to Bitcoin halving for example)
- or do not set `esLBR` max supply (still do your best to limit it to `100 000 000`, but if it goes above that number, users will still be able to claim their acquired rewards).


## Assessed type

ERC20