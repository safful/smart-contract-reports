## Tags

- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- MR-NEW

# [Hard slippage in `Reth.withdraw()`](https://github.com/code-423n4/2023-05-asymmetry-mitigation-findings/issues/67) 

# Hard slippage in Reth.withdraw()

https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/Reth.sol#L121

## Description
A hard slippage has been introduced [in `Reth.withdraw()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/Reth.sol#L121). This is a new occurrence of part of M-12 (not the main report, but e.g. [this duplicate](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/699)), namely that the slippage can be changed only by the owner, which under volatile market conditions or a depegging may be violated and thus DoS unstaking.

Note that the aspect of this issue that a user may lose funds because of an undesirable slippage which he cannot change has been fixed by the mitigation of M-12. The aspect detailed here, however, has not been fixed, and this is a new occurrence of the same type.

## Recommendation
Remove all slippage control from the derivatives and control slippage only in `SafEth.unstake()` and `SafEth.unstake()` with the new `_minOut` which was put in place in the (unsuccessful) mitigation of M-12. Note that this is the same fix as for what remains to fix in M-12.