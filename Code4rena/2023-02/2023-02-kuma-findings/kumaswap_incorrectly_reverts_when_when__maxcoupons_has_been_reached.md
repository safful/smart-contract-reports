## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-04

# [KUMASwap incorrectly reverts when when _maxCoupons has been reached](https://github.com/code-423n4/2023-02-kuma-findings/issues/10) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMASwap.sol#L115-L171


# Vulnerability details

## Impact

Selling bonds with coupons that are already accounted will fail unexpectedly

## Proof of Concept

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMASwap.sol#L116-L118

        if (_coupons.length() == _maxCoupons) {
            revert Errors.MAX_COUPONS_REACHED();
        }

The above lines will cause ALL bonds sales to revert when `_coupons.length` has reached `_maxCoupons`. Since bonds may share the same `coupon`, the swap should continue to accept bonds with a `coupon` that already exist in the `_coupons` set.

## Tools Used

Manual Review

## Recommended Mitigation Steps

sellBond should only revert if the max length has been reached and bond.coupon doesn't already exist:

    -   if (_coupons.length() == _maxCoupons) {
    +   if (_coupons.length() == _maxCoupons && !_coupons.contains(bond.coupon)) {
            revert Errors.MAX_COUPONS_REACHED();
        }