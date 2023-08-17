## Tags

- 2 (Med Risk)
- satisfactory
- selected for report
- MR-NEW

# [[term-fix] Mitigation Error](https://github.com/code-423n4/2023-03-kuma-mitigation-contest-findings/issues/26) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/22fd56b3f0df71714cb71f1ce2585f1c4dd21d64/src/kuma-protocol/KUMASwap.sol#L177-L220


# Vulnerability details

Note - The term refactoring has been made for the following reason:

`Our main KIBT is intended to be backed by 1-year treasury bill tokens, however, a bond issued on 1 Jan 2023 does not have the same amount of seconds compared to a 1-year treasury bill issued on 1 March 2023, or 1 Jan 2024 etc.`

This seems to be a misunderstanding because a 1-year treasury bill is not 1 year in length. It's actually a 52 week fixed length bond ([source](https://www.treasurydirect.gov/marketable-securities/treasury-bills/)), so there's no need for a change. Second as explained below using the new system will cause compatibility issues for variable length bonds like treasury notes and other non-US bonds.

## Impact

Non-fixed terms will cause variable length bonds with the same nominal yield to have different coupons

## Proof of Concept

Currently coupon is used to track which bonds can and cannot be sold to KUMASwap. Coupon is the amount of yield the bond accumulates per second. Since term is now tracked in months rather than seconds the coupon rate will have to be different between nearly every bond issued. It will also hurt the compatibility of bonds because even when the underlying bond rate doesn't change, the coupon will have to be adjusted. 

Example:
There are 31536000 seconds in a calendar year and there are 31622400 seconds in a leap year. This means that bonds that include the end of a leap year will have to have a different (and lower) coupon than bonds with the same yield that don't include the end of a leap year. This is because bonds that do will have a longer term and even though their yields are the same nominally (i.e. 3.5%) they're coupons will be different. 

3.5% (non-leap year) => coupon = 1.00000000109085
3.5% (leap year) => coupon = 1.00000000108785

The result of this is that bonds that share the same nominal yields will be forced to have different coupons or else their value will be calculated incorrectly. This causes the frustrating problem that two bonds with the same nominal yield will have different coupons if one includes the end of a leap year while the other doesn't. Since bonds that have a coupon lower than the current rate can't be sold to KUMA swap these bonds can't be sold. This becomes increasingly complicated (and more fragmented) the longer the term of the bonds because different issuance dates will includes more or less leap year ends.

It is also worth considering that different types of bonds will behave differently. As an example T-Bills are fixed length bonds. The "1 year" T-Bill is actually a 52 week fixed length bond. The "2 year" T-Notes are variable length bonds which depends on leap years

## Tools Used

Manual Review

## Recommended Mitigation Steps

Even though some bonds should have slightly longer terms (because of leap years) the previous way a tracking using fixed term length in seconds will provide a better user experience and prevent bonds with the same nominal yield from having different coupons.