## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [`LimboDAO.seed`: Wrong error message](https://github.com/code-423n4/2022-01-behodler-findings/issues/167) 

# Handle

cmichel


# Vulnerability details

The error message for the `uniLPs` is still referring to `Sushi` instead of `Uniswap`

```solidity
require(UniPairLike(uniLPs[i]).factory() == uniFactory, "LimboDAO: invalid Sushi LP");
```

