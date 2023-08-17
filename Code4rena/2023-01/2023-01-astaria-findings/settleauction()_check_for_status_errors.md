## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [settleAuction() Check for status errors](https://github.com/code-423n4/2023-01-astaria-findings/issues/582) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L526-L534


# Vulnerability details

## Impact
ClearingHouse.safeTransferFrom() to execute successfully even if there is no bid

## Proof of Concept
settleAuction is called at the end of the auction and will check if the status is legal

```solidity

  function settleAuction(uint256 collateralId) public {
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

This check seems to be miswritten，The normal logic would be

```solidity
s.collateralIdToAuction[collateralId] == bytes32(0) || ERC721(s.idToUnderlying[collateralId].tokenContract).ownerOf(
        s.idToUnderlying[collateralId].tokenId
      ) == s.clearingHouse[collateralId]
```

This causes ClearingHouse.safeTransferFrom() to execute successfully even if there is no bid

## Tools Used

## Recommended Mitigation Steps

```solidity

  function settleAuction(uint256 collateralId) public {
    if (
-     s.collateralIdToAuction[collateralId] == bytes32(0) &&
-    ERC721(s.idToUnderlying[collateralId].tokenContract).ownerOf(
-        s.idToUnderlying[collateralId].tokenId
-      ) !=
-      s.clearingHouse[collateralId]
+      s.collateralIdToAuction[collateralId] == bytes32(0) || 
+       ERC721(s.idToUnderlying[collateralId].tokenContract).ownerOf(s.idToUnderlying[collateralId].tokenId
+      ) == 
+      s.clearingHouse[collateralId]
    ) {
      revert InvalidCollateralState(InvalidCollateralStates.NO_AUCTION);
    }
```