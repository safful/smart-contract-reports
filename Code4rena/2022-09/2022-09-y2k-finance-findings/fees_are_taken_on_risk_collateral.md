## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- edited-by-warden
- selected for report

# [Fees are taken on risk collateral](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/44) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/Vault.sol#L203-L234


# Vulnerability details

## Impact

Fees are taken on funds deposited as collateral

## Proof of Concept

    uint256 feeValue = calculateWithdrawalFeeValue(entitledShares, id);

In L226 of Vault.sol#withdraw the fee is taken on the entire collateral deposited by the risk users. This is problematic for two reasons. The first is that the collateral provided by the risk users will likely be many many times higher than the premium being paid by the hedge users. This will create a strong disincentive to use the protocol because it is likely a large portion of the profits will be taking by fees and the risk user may unexpectedly lose funds overall if premiums are too low.

The second issue is that this method of fees directly contradicts project [documents](https://medium.com/@Y2KFinance/introducing-earthquake-pt-2-6f206cd4b315) which clearly indicate that fees are only taken on the premium and insurance payouts, not when risk users are receiving their collateral back. 

## Tools Used

## Recommended Mitigation Steps

Fee calculations should be restructured to only take fees on premiums and insurance payouts