## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-13

# [Anyone can wipe complete state of any collateral at any point](https://github.com/code-423n4/2023-01-astaria-findings/issues/287) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L114-L167
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L524-L545
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L497-L510
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L623-L656


# Vulnerability details

## Impact
The Clearing House is implemented as an ERC1155. This is used to settle up at the end of an auction. The Clearing House's token is listed as one of the Consideration Items, and when Seaport goes to transfer it, it triggers the settlement process. 

This settlement process includes deleting the collateral state hash from LienToken.sol, burning all lien tokens, deleting the idToUnderlying mapping, and burning the collateral token. **These changes effectively wipe out all record of the liens, as well as removing any claim the borrower has on their underlying collateral.**

After an auction, this works as intended. The function verifies that sufficient payment has been made to meet the auction criteria, and therefore all these variables should be zeroed out. 

However, the issue is that there is no check that this safeTransferFrom function is being called after an auction has completed. In the case that it is called when there is no auction, all the auction criteria will be set to 0, and therefore the above deletions can be performed with a payment of 0.

This allows any user to call the `safeTransferFrom()` function for any other user's collateral. This will wipe out all the liens on that collateral, and burn the borrower's collateral token, and with it their ability to ever reclaim their collateral. 

## Proof of Concept

The flow is as follows:
- safeTransferFrom(offerer, buyer, paymentToken, amount, data)
- _execute(offerer, buyer, paymentToken, amount)
- using the auctionStack in storage, it calculates the amount the auction would currently be listed at
- it confirms that the Clearing House has already received sufficient paymentTokens for this amount
- it then transfers the liquidator their payment (currently 13%)
- it calls `LienToken#payDebtViaClearingHouse()`, which pays back all liens, zeros out all lien storage and deletes the collateralStateHash
- if there is any remaining balance of paymentToken, it transfers it to the owner of the collateral
- it then calls `Collateral#settleAuction()`, which deletes idToUnderlying, collateralIdToAuction and burns the collateral token

In the case where the auction hasn't started, the `auctionStack` in storage is all set to zero. When it calculates the payment that should be made, it uses `_locateCurrentAmount`, which simply returns `endAmount` if `startAmount == endAmount`. In the case where they are all 0, this returns 0.

The second check that should catch this occurs in `settleAuction()`:

```
    if (
      s.collateralIdToAuction[collateralId] == bytes32(0) &&
      ERC721(s.idToUnderlying[collateralId].tokenContract).ownerOf(
        s.idToUnderlying[collateralId].tokenId
      ) !=
      s.clearingHouse[collateralId]
    ) {
      revert InvalidCollateralState(InvalidCollateralStates.NO_AUCTION);
    }
```

However, this check accidentally uses an `&&` operator instead of a `||`. The result is that, even if the auction hasn't started, only the first criteria is false. The second is checking whether the Clearing House owns the underlying collateral, which happens as soon as the collateral is deposited in `CollateralToken.sol#onERC721Received()`:

```
      ERC721(msg.sender).safeTransferFrom(
        address(this),
        s.clearingHouse[collateralId],
        tokenId_
      );
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Change the check in `settleAuction()` from an AND to an OR, which will block any collateralId that isn't currently at auction from being settled: 

```
    if (
      s.collateralIdToAuction[collateralId] == bytes32(0) ||
      ERC721(s.idToUnderlying[collateralId].tokenContract).ownerOf(
        s.idToUnderlying[collateralId].tokenId
      ) !=
      s.clearingHouse[collateralId]
    ) {
      revert InvalidCollateralState(InvalidCollateralStates.NO_AUCTION);
    }
```