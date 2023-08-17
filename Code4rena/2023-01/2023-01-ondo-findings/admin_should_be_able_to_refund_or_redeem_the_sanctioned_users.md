## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor disputed
- primary issue
- M-01

# [Admin should be able to refund or redeem the sanctioned users](https://github.com/code-423n4/2023-01-ondo-findings/issues/265) 

# Lines of code

https://github.com/code-423n4/2023-01-ondo/blob/f3426e5b6b4561e09460b2e6471eb694efdd6c70/contracts/cash/CashManager.sol#L707


# Vulnerability details

## Impact
Sanctioned user's funds are locked

## Proof of Concept
It is understood that the sanctioned users can not mint nor redeem because the functions `requestMint()` and `requestRedemption()` are protected by the modifier `checkKYC()`.
And it is also understood that the protocol team knows about this.
But I still believe the admin should be able to refund or redeem those funds.
And it is not possible for now because the KYC is checked for the `redeemers` and `refundees` in the function `completeRedemptions()`.
So as long as the user becomes unverified (due to several reasons including the signature expiry), the funds are completely locked and even the admin has no control over it.

```solidity
CashManager.sol
707:   function completeRedemptions(
708:     address[] calldata redeemers,
709:     address[] calldata refundees,
710:     uint256 collateralAmountToDist,
711:     uint256 epochToService,
712:     uint256 fees
713:   ) external override updateEpoch onlyRole(MANAGER_ADMIN) {
714:     _checkAddressesKYC(redeemers);
715:     _checkAddressesKYC(refundees);
716:     if (epochToService >= currentEpoch) {
717:       revert MustServicePastEpoch();
718:     }
719:     // Calculate the total quantity of shares tokens burned w/n an epoch
720:     uint256 refundedAmt = _processRefund(refundees, epochToService);
721:     uint256 quantityBurned = redemptionInfoPerEpoch[epochToService]
722:       .totalBurned - refundedAmt;
723:     uint256 amountToDist = collateralAmountToDist - fees;
724:     _processRedemption(redeemers, amountToDist, quantityBurned, epochToService);
725:     collateral.safeTransferFrom(assetSender, feeRecipient, fees);
726:     emit RedemptionFeesCollected(feeRecipient, fees, epochToService);
727:   }

```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Assuming that the `MANAGER_ADMIN` can be trusted, I suggest removing KYC check for the redeemers and refundees.