## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-04

# [Unstaking does not update the mapping `sETHUserClaimForKnot`](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/90) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L245


# Vulnerability details

## Impact

If a user stakes some sETH, and after some time decides to unstake some amount of sETH, later s/he will not be qualified or be less qualified to claim ETH on the remaining staked sETH.

## Proof of Concept

Suppose Alice stakes 5 sETH by calling `stake(...)`.
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L203
So, we will have:
 -  `sETHUserClaimForKnot[BLS][Alice] = (5 * 10^18 * accumulatedETHPerFreeFloatingShare) / PRECISION`
 - `sETHStakedBalanceForKnot[BLS][Alice] = 5 * 10^18`
 - `sETHTotalStakeForKnot[BLS] += 5 * 10^18`

Later, Alice decides to unstake 3 sETH by calling `unstake(...)`.
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L245

So, all ETH owed to Alice will be paid:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L257

Then, we will have:
 -  `sETHUserClaimForKnot[BLS][Alice] = (5 * 10^18 * accumulatedETHPerFreeFloatingShare) / PRECISION`
 - `sETHStakedBalanceForKnot[BLS][Alice] = 2 * 10^18`
 - `sETHTotalStakeForKnot[BLS] -= 3 * 10^18`

It is clear that the mapping `sETHStakedBalanceForKnot` is decreased as expected, but the mapping `sETHUserClaimForKnot` is not changed. In other words, the mapping `sETHUserClaimForKnot` is still holding the claimed amount based on the time 5 sETH were staked.

If, after some time, the ETH is accumulated per free floating share for the BLS public key that Alice was staking for, Alice will be qualified to some more ETH to claim (because she has still 2 sETH staked). 

If Alice unstakes by calling `unstake(...)` or claim ETH by calling `claimAsStaker(...)`, in both calls, the function `calculateUnclaimedFreeFloatingETHShare` will be called to calculate the amount of unclaimed ETH:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L652

In this function, we will have:
 - `stakedBal = sETHStakedBalanceForKnot[BLS][Alice]` = 2 * 10^18
 - `userShare = (newAccumulatedETHPerShare * stakedBal) / PRECISION`
 
The return value which is unclaimed ETH will be:
```
userShare - sETHUserClaimForKnot[BLS][Alice] = 
(newAccumulatedETHPerShare * 2 * 10^18) / PRECISION - (5 * 10^18 * accumulatedETHPerFreeFloatingShare) / PRECISION
```

This return value is not correct (it is highly possible to be smaller than 0, and as a result Alice can not claim anything), because the claimed ETH is still based on the time when 5 sETH were staked, not on the time when 2 sETH were remaining/staked.

The vulnerability is that during unstaking, the mapping `sETHUserClaimForKnot` is not updated to the correct value. In other words, this mapping is updated in `_claimAsStaker`, but it is updated based on 5 sETH staked, later when 3 sETH are unstaked, this mapping should be again updated based on the remaing sETH (which is 2 sETH).

As a result, Alice can not claim ETH or she will qualify for less amount.

## Tools Used

## Recommended Mitigation Steps
The following line should be added on line 274:
```
sETHUserClaimForKnot[_blsPubKey][msg.sender] =
                (accumulatedETHPerShare * sETHStakedBalanceForKnot[_blsPubKey][msg.sender]) / PRECISION
```

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/syndicate/Syndicate.sol#L274