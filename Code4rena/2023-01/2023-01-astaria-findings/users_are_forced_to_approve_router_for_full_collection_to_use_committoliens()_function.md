## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-19

# [Users are forced to approve Router for full collection to use commitToLiens() function](https://github.com/code-423n4/2023-01-astaria-findings/issues/283) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L780-L785
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L287-L306
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L233-L244


# Vulnerability details

## Impact
When a user calls Router#commitToLiens(), the Router calls `commitToLien()`. The comments specify:

`//router must be approved for the collateral to take a loan,`

However, the Router being approved isn't enough. It must be approved for all, which is a level of approvals that many users are not comfortable with. This is because, when the commitment is validated, it is checked as follows:

```
    uint256 collateralId = params.tokenContract.computeId(params.tokenId);
    ERC721 CT = ERC721(address(COLLATERAL_TOKEN()));
    address holder = CT.ownerOf(collateralId);
    address operator = CT.getApproved(collateralId);
    if (
      msg.sender != holder &&
      receiver != holder &&
      receiver != operator &&
      !CT.isApprovedForAll(holder, msg.sender)
    ) {
      revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
    }
```

## Proof of Concept

The check above allows the following situations pass:
- caller of the function == owner of the NFT
- receiver of the loan == owner of the NFT
- receiver of the loan == address approved for the individual NFT
- caller of the function == address approved for all

This is inconsistent and doesn't make much sense. The approved users should have the same permissions. 

More importantly, the most common flow (that the address approved for the individual NFT — the Router — is the caller) does not work and will lead to the function reverting.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Change the check to include `msg.sender != operator` rather than `receiver != operator`.