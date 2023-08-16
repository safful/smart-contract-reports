## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Gov.sol: Non-intuitive comment in tokenRemove()](https://github.com/code-423n4/2021-07-sherlock-findings/issues/33) 

# Handle

hickuphh3


# Vulnerability details

### Impact

In `tokenRemove()`, the comment `// NOTE: removed because firstMoneyOut will always be less or equal to stakeBalance` is not intuitive because it is not clear what is removed.

Perhaps `// NOTE: check that firstMoneyOut == 0 not needed since firstMoneyOut <= stakeBalance` will be better.

