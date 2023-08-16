## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Unused Named Return](https://github.com/code-423n4/2021-11-nested-findings/issues/105) 

# Handle

ye0lde


# Vulnerability details

## Impact

Removing unused named return variables can reduce gas usage and improve code clarity.

## Proof of Concept

Unused named return:
https://github.com/code-423n4/2021-11-nested/blob/f646002b692ca5fa3631acfff87dda897541cf41/contracts/NestedFactory.sol#L69

## Tools Used
Visual Studio Code, Remix

## Recommended Mitigation Steps
Remove the unused named return

