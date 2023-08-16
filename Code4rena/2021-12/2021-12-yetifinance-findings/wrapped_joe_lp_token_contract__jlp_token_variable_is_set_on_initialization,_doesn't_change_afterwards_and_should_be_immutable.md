## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Wrapped Joe LP token Contract  JLP token variable is set on initialization, doesn't change afterwards and should be immutable](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/69) 

# Handle

Jujic


# Vulnerability details

## Impact
```
IERC20 public immutable JLP;
IERC20 public  immutable JOE;
```

## Proof of Concept
https://github.com/code-423n4/2021-12-yetifinance/blob/5f5bf61209b722ba568623d8446111b1ea5cb61c/packages/contracts/contracts/AssetWrappers/WJLP/WJLP.sol#L41-L44

## Tools Used
REmix
## Recommended Mitigation Steps

