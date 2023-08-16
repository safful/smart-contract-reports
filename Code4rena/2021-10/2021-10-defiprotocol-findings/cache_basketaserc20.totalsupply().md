## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Cache basketAsERC20.totalSupply()](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/88) 

# Handle

pauliax


# Vulnerability details

## Impact
Here basketAsERC20.totalSupply() does not change inside the loop so it can be called outside the loop to avoid multiple duplicate external calls:
  uint256 tokensNeeded = basketAsERC20.totalSupply() * pendingWeights[i] * newRatio / BASE / BASE;

## Recommended Mitigation Steps
Cache basketAsERC20.totalSupply() in a temporary variable and re-use it.

