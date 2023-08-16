## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- resolved
- sponsor confirmed

# [Outdated comment in `TimelockRewardDistributionTokenImpl.burnFrom`](https://github.com/code-423n4/2021-12-nftx-findings/issues/150) 

# Handle

cmichel


# Vulnerability details

The comment in `TimelockRewardDistributionTokenImpl.burnFrom` says:
> the caller must have allowance for ``accounts``'s tokens of at least `amount`.

This was the case in a previous version but not anymore. The owner does not need to be approved to burn tokens anymore.

## Recommended Mitigation Steps
Update the comment to clarify the behavior.


