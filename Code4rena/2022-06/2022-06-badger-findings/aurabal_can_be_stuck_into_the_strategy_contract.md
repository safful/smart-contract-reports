## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- valid

# [auraBAL can be stuck into the Strategy contract](https://github.com/code-423n4/2022-06-badger-findings/issues/129) 

# Lines of code

https://github.com/Badger-Finance/vested-aura/blob/v0.0.2/contracts/MyStrategy.sol#L220-L228
https://github.com/Badger-Finance/vested-aura/blob/v0.0.2/contracts/MyStrategy.sol#L288


# Vulnerability details

## Impact
The internal `_harvest()` function defined is responsible to claim auraBAL from the aura locker and within the function it swaps them to auraBAL -> BAL/ETH BPT -> WETH -> AURA, finally it locks AURA to the locker to increase the position. For claiming auraBAL it calls `LOCKER.getReward(address(this))` and it calculates the tokes earned, checking the balance before and after the claiming. 
The function to get the rewards is public and any address can call it for the strategy address, and it will transfer all rewards tokens to the strategy, but in this scenario the auraBAL will remain in stuck into the contract, because they won't be counted as auraBAL earned during the next `_harvest()`. Also they could not sweep because auraBAL is a protected token. 
Also, the aura Locker will be able to add other token as reward apart of auraBAL, but the harvest function won't be able to manage them, so they will need to be sweep every time.

The same scenario can happen during the `claimBribesFromHiddenHand()` call, the `IRewardDistributor.Claim[] calldata _claims` pass as input parameters could be frontrunned, and another address can call the `hiddenHandDistributor.claim(_claims)` (except for ETH rewards) for the strategy address, and like during the `_harvest()` only the tokens received during the call will be counted as earned. However every token, except auraBAL can be sweep, but the `_notifyBribesProcessor()` may never be called.

## Proof of Concept
At every `_harvest()` it checks the balance before the claim and after, to calculate the auraBAL earned, so every auraBAL transferred to the strategy address not during this call, won't be swapped to AURA. 

## Recommended Mitigation Steps
Instead of calculating the balance before and after the claim, for both `harvest≠ and `claimBribesFromHiddenHand()`, the whole balance could be taken, directly after the claim.

