## Tags

- bug
- question
- 2 (Med Risk)
- resolved
- sponsor confirmed

# [user won't be able to get his rewards in case of staking with amount = 0](https://github.com/code-423n4/2022-01-behodler-findings/issues/146) 

# Handle

CertoraInc


# Vulnerability details

## Limbo.sol (`stake()` function)
if a user has a pending reward and he call the `stake` function with `amount = 0`, he won't be able to get his reward (he won't get the reward, and the reward debt will cover the reward)

that's happening because the reward calculation is done only if the staked amount (given as a parameter) is greater than 0, and it updates the reward debt also if the amount is 0, so the reward debt will be updated without the user will be able to get his reward

