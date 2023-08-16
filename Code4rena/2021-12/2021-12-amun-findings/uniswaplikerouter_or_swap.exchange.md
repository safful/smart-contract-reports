## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [uniSwapLikeRouter or swap.exchange](https://github.com/code-423n4/2021-12-amun-findings/issues/286) 

# Handle

pauliax


# Vulnerability details

## Impact
Contracts SingleTokenJoinV2 and SingleNativeTokenExitV2 initialize uniSwapLikeRouter, but never actually use it, as swap.exchange is used instead. So it basically trusts the user input. Consider removing uniSwapLikeRouter if that was the intention to save some gas.

