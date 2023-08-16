## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [typo: totalAccountBorrrow](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/130) 

# Handle

0xsanson


# Vulnerability details

## Impact
Simple typo: totalAccountBorrrow instead of totalAccountBorrow

## Proof of Concept
In LendingPair.sol:
```
    uint totalAccountBorrrow = _borrowBalance(_account, tokenA, tokenA) + _borrowBalance(_account, tokenB, tokenA);

    return totalAccountSupply * 1e18 / totalAccountBorrrow;
```

## Tools Used
Manual analysis

## Recommended Mitigation Steps
Correct the typo

