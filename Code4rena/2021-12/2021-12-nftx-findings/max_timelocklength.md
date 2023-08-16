## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [max timelockLength](https://github.com/code-423n4/2021-12-nftx-findings/issues/172) 

# Handle

pauliax


# Vulnerability details

## Impact
Consider introducing a reasonable global upper limit for timelockLength in XTokenUpgradeable and TimelockRewardDistributionTokenImpl, so the users can't be locked out of their tokens forever.

## Recommended Mitigation Steps
XTokenUpgradeable and TimelockRewardDistributionTokenImpl should not trust the external input but have explicitly declared boundaries for values like timelock length to reduce possibilities of unexpected outcomes.

