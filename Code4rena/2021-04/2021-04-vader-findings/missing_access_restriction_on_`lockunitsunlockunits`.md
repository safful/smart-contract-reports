## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- filed

# [Missing access restriction on `lockUnits/unlockUnits`](https://github.com/code-423n4/2021-04-vader-findings/issues/208) 

# Handle

@cmichelio


# Vulnerability details

## Vulnerability Details

The `Pool.lockUnits` allows anyone to steal pool tokens from a `member` and assign them to `msg.sender`.

## Impact

Anyone can steal pool tokens from any other user.

## Recommended Mitigation Steps

Add access control and require that `msg.sender` is the router or another authorized party.


