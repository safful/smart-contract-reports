## Tags

- bug
- 2 (Med Risk)
- satisfactory
- sponsor acknowledged
- selected for report
- M-07

# [Oracle's two-day feature can be gamed](https://github.com/code-423n4/2022-10-inverse-findings/issues/278) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L124


# Vulnerability details

## Impact
The two-day feature of the oracle can be gamed where you only have to manipulate the oracle for ~2 blocks.

## Proof of Concept
The oracle computes the day using:
```sol
uint day = block.timestamp / 1 days;
```

Since we're working with `uint` values here, the following is true:
$1728799 / 86400 = 1$
$172800 / 86400 = 2$

Meaning, if you manipulate the oracle at the last block of day $X$, e.g. 23:59:50, and at the first block of day $X + 1$, e.g. 00:00:02, you bypass the two-day feature of the oracle. You only have to manipulate the oracle for two blocks.

This is quite hard to pull off. I'm also not sure whether there were any instances of Chainlink oracle manipulation before. But, since you designed this feature to prevent small timeframe oracle manipulation I think it's valid to point this out.

## Tools Used
none

## Recommended Mitigation Steps
If you increase it to a three-day interval you can fix this issue. Then, the oracle has to be manipulated for at least 24 hours.