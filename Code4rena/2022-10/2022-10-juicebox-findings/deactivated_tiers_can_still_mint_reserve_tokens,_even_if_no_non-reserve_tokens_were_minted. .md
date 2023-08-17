## Tags

- bug
- 2 (Med Risk)
- sponsor acknowledged
- satisfactory
- selected for report
- M-07

# [Deactivated tiers can still mint reserve tokens, even if no non-reserve tokens were minted. ](https://github.com/code-423n4/2022-10-juicebox-findings/issues/189) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721DelegateStore.sol#L808


# Vulnerability details

## Description

Tiers in Juicebox can be deactivated using the adjustTiers() function. It makes sense that reserve tokens may be minted in deactivated tiers, in order to be consistent with already minted tokens. However, the code allows the first reserve token to be minted in a deactivated tier, *even* though there was no previous minting of that tier.

```
function recordMintReservesFor(uint256 _tierId, uint256 _count)
  external
  override
  returns (uint256[] memory tokenIds)
{
  // Get a reference to the tier.
  JBStored721Tier storage _storedTier = _storedTierOf[msg.sender][_tierId];
  // Get a reference to the number of reserved tokens mintable for the tier.
  uint256 _numberOfReservedTokensOutstanding = _numberOfReservedTokensOutstandingFor(
    msg.sender,
    _tierId,
    _storedTier
  );
  if (_count > _numberOfReservedTokensOutstanding) revert INSUFFICIENT_RESERVES();
  ...
  for (uint256 _i; _i < _count; ) {
  // Generate the tokens.
  tokenIds[_i] = _generateTokenId(
    _tierId,
    _storedTier.initialQuantity - --_storedTier.remainingQuantity + _numberOfBurnedFromTier
  );
  unchecked {
    ++_i;
  }
}
```

```
function _numberOfReservedTokensOutstandingFor(
  address _nft,
  uint256 _tierId,
  JBStored721Tier memory _storedTier
) internal view returns (uint256) {
  // Invalid tier or no reserved rate?
  if (_storedTier.initialQuantity == 0 || _storedTier.reservedRate == 0) return 0;
  // No token minted yet? Round up to 1.
  // ******************* BUG HERE *********************
  if (_storedTier.initialQuantity == _storedTier.remainingQuantity) return 1;
  ...
}
```

Using the rounding mechanism is not valid when the tier has been deactivated, since we know there won't be any minting of this tier.

## Impact

The reserve beneficiary receives an unfair NFT which may be used to withdraw tokens using the redemption mechanism.

## Tools Used

Manual audit

## Recommended Mitigation Steps

If Juicebox intends to use rounding functionality, pass an argument *isDeactivated* which, if true, deactivated the rounding logic.