## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor disputed
- M-02

# [A user can 'borrow' dMute balance for a single block to increase their amplifier APY](https://github.com/code-423n4/2023-03-mute-findings/issues/36) 

# Lines of code

https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L482-L486


# Vulnerability details



The amplifier's APY is calculated based on the user's dMute balance (delegation balance to be more accurate) - the more dMute the user holds the higher APY they get.
However, the contract only checks the user's dMute balance at staking, the user doesn't have to hold that balance at any time but at the staking block.

This let's the user to bribe somebody to delegate them their dMute for a single block and stake at the same time.

Since the balance checked is the delegation balance (`getPriorVotes`) this even makes it easier - since the 'lender' doesn't even need to trust the 'borrower' to return the funds, all the 'lender' can cancel the delegation on their own on the next block.

The lender can also create a smart contract to automate and decentralize this 'service'.


## Impact
Users can get a share of the rewards that are supposed to incentivize them to hold dMute without actually holding them.
In other words, the same dMute tokens can be used to increase simultaneous staking of different users.

## Proof of Concept

[The following code](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L482-L486) shows that only a single block is being checked to calculate the `accountDTokenValue` at `calculateMultiplier()`:
```solidity
        if(staked_block != 0 && enforce)
          accountDTokenValue = IDMute(dToken).getPriorVotes(account, staked_block);
        else
          accountDTokenValue = IDMute(dToken).getPriorVotes(account, block.number - 1);
```

The block checked when rewarding is the staked block, since `enforce` is true [when applying reward](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L371):
```
        reward = lpTokenOut.mul(_totalWeight.sub(_userWeighted[account])).div(calculateMultiplier(account, true));
```



## Recommended Mitigation Steps
Make sure that the user holds the dMute for a longer time, one way to do it might be to sample a few random blocks between staking and current blocks and use the minimum balance as the user's balance.