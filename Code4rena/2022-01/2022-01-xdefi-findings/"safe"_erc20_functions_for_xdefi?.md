## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# ["Safe" ERC20 functions for XDEFI?](https://github.com/code-423n4/2022-01-xdefi-findings/issues/194) 

# Handle

0xsanson


# Vulnerability details

## Impact
Throughout the code the safe functions `safeTransfer` and `safeTransferFrom` are used when dealing with XDEFI. Isn't this token a standard ERC20? I believe the normal ERC20 transfer functions can be used. The advantage is gaining some 100s gas otherwise spent in unneeded logic.

## Proof of Concept
grep safeT *.sol

## Recommended Mitigation Steps
Consider removing the SafeERC20 library.

