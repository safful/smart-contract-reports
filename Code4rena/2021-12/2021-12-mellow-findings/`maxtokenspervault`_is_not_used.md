## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`maxTokensPerVault` is not used](https://github.com/code-423n4/2021-12-mellow-findings/issues/65) 

# Handle

0x0x0x


# Vulnerability details

In `protocolGovernance.sol`, there is parameter `maxTokensPerVault`. This parameter is never utilized, therefore does not provide the functionality stated in comments.

