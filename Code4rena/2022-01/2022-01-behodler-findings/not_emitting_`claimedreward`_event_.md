## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [not emitting `ClaimedReward` event ](https://github.com/code-423n4/2022-01-behodler-findings/issues/148) 

# Handle

CertoraInc


# Vulnerability details

## Limbo.sol (`_unstake()` and `stake()` functions) 
not emitting the `ClaimedReward` event when the user claims his rewards (also when staking and getting the current reward, I don't know if it is done in purpose but just making sure)

