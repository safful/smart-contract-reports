## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-26

# [CollateralToken should allow to execute token owner's action to approved addresses](https://github.com/code-423n4/2023-01-astaria-findings/issues/134) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L274
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/CollateralToken.sol#L266
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L238


# Vulnerability details

## Impact

CollateralToken should allow to execute token owner's action to approved addresses

## Proof of Concept

Let us look into the code below in VaultImplementation#_validateCommitment

```solidity
function _validateCommitment(
IAstariaRouter.Commitment calldata params,
address receiver
) internal view {
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

the code check should also check that msg.sender != operator to make the check complete, if the msg.sender comes from an approved operator, the call should be valid

```solidity
if (
  msg.sender != holder &&
  receiver != holder &&
  receiver != operator &&
  msg.sender != operator &&
  !CT.isApprovedForAll(holder, msg.sender)
) {
  revert InvalidRequest(InvalidRequestReason.NO_AUTHORITY);
}
```

AND

CollateralToken functions flashAction, releaseToAddress are restricted to the owner of token only. But they should be allowed for approved addresses as well.

For example, in flashAuction, only the owner of the collateral token can start the flashAction, then approved operator by owner cannot start flashAction

```solidity
  function flashAction(
    IFlashAction receiver,
    uint256 collateralId,
    bytes calldata data
  ) external onlyOwner(collateralId) {
```

note the check onlyOwner(collateralId) does not check if the msg.sender is an approved operator.

```solidity
modifier onlyOwner(uint256 collateralId) {
	require(ownerOf(collateralId) == msg.sender);
	_;
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add ability for approved operators to call functions that can be called by the collateral token owner.