## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Incorrect Info in Comment in Alchemist.sol (138)](https://github.com/code-423n4/2021-11-yaxis-findings/issues/6) 

# Handle

TimmyToes


# Vulnerability details

## Impact
Developers wishing to interact with yAxis will find it harder to do so.

## Proof of Concept
Lines 138 of Alchemist.sol
/// @dev The percent of each profitable harvest that will go to the rewards contract.
This comment is incorrect. The borrow fee is charged on mint against debt, not harvest.

## Recommended Mitigation Steps
Edit the comment.


