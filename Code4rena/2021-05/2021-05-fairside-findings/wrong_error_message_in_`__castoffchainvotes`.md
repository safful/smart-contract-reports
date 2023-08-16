## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- disagree with severity
- resolved

# [Wrong error message in `__castOffchainVotes`](https://github.com/code-423n4/2021-05-fairside-findings/issues/36) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

The error message states:

```solidity
require(
    proposal.offchain,
    "FairSideDAO::__castOffchainVotes: proposal is meant to be voted offchain"
);
```

But it should be "... meant to be voted onchain".


