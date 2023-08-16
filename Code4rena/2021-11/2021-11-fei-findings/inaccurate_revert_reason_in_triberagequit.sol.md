## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Inaccurate revert reason in TRIBERagequit.sol](https://github.com/code-423n4/2021-11-fei-findings/issues/122) 

# Handle

gzeon


# Vulnerability details

## Impact
The revert reason in L74 is inaccurate. It should be "ragequit more than assigned".
L72-75
```
        require(
            (claimed[thisSender] + multiplier) <= key,
            "already ragequit all you tokens"
        );
```
The original revert string wrongly implies that claimed[thisSender] >= key
Might cause confusing when user who have not yet claimed mistakenly supplied a multiplier > key

