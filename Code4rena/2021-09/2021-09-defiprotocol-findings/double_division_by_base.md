## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Double division by BASE](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/210) 

# Handle

pauliax


# Vulnerability details

## Impact
This double division by BASE can be eliminated to improve precision and reduce gas costs:
   uint256 tokensNeeded = basketAsERC20.totalSupply() * pendingWeights[i] * newRatio / BASE / BASE;

## Recommended Mitigation Steps
if you introduce a constant variable, e.g.:
   uint256 private constant BASE_2X = BASE * 2;
   uint256 tokensNeeded = basketAsERC20.totalSupply() * pendingWeights[i] * newRatio / BASE_2X;

