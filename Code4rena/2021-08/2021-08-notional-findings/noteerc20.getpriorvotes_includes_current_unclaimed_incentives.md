## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [NoteERC20.getPriorVotes includes current unclaimed incentives](https://github.com/code-423n4/2021-08-notional-findings/issues/76) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `NoteERC20.getPriorVotes` function is supposed to return the voting strength of an account at a specific block in the past.
This should be a static value but it directly includes the _current_ unclaimed incentives due to the `getUnclaimedVotes(account)` call.

## Impact
Users that didn't even have tokens at the time of proposal creation but are now interested in voting on the proposal can farm unclaimed incentives and impact the outcome of the proposal.

## Recommended Mitigation Steps
Adding checkpoints for all unclaimed incentives would be the correct solution but was probably not done because it'd cost too much gas.
It also needs to be ensured that incentives cannot be increased through flash-loaning of assets.

