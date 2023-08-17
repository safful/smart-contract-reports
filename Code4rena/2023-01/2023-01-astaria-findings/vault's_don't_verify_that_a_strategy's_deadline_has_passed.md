## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-19

# [Vault's don't verify that a strategy's deadline has passed](https://github.com/code-423n4/2023-01-astaria-findings/issues/122) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/VaultImplementation.sol#L229-L266
https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L439


# Vulnerability details

## Impact
The vault doesn't verify that a deadline hasn't passed when a commitment is validated. Users are able to take out loans using strategies that have already expired. Depending on the nature of the strategy that can cause a loss of funds for the LPs.

## Proof of Concept
When you take out a loan using the AstariaRouter, the deadline is verified:
```sol
  function _validateCommitment(
    RouterStorage storage s,
    IAstariaRouter.Commitment calldata commitment,
    uint256 timeToSecondEpochEnd
  ) internal view returns (ILienToken.Lien memory lien) {
    if (block.timestamp > commitment.lienRequest.strategy.deadline) {
      revert InvalidCommitmentState(CommitmentState.EXPIRED);
    }
// ...
```
But, `VaultImplementation._validateCommitment()` skips that check:
```sol
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
    VIData storage s = _loadVISlot();
    address recovered = ecrecover(
      keccak256(
        _encodeStrategyData(
          s,
          params.lienRequest.strategy,
          params.lienRequest.merkle.root
        )
      ),
      params.lienRequest.v,
      params.lienRequest.r,
      params.lienRequest.s
    );
    if (
      (recovered != owner() && recovered != s.delegate) ||
      recovered == address(0)
    ) {
      revert IVaultImplementation.InvalidRequest(
        InvalidRequestReason.INVALID_SIGNATURE
      );
    }
  }
```

If you search for `deadline` in the codebase you'll see that there's no other place where the property is accessed.

As long as the user takes out the loan from the vault directly, they can use strategies that have expired. The vault owner could prevent this from happening by incrementing the `strategistNonce` after the strategy expired.

## Tools Used
none

## Recommended Mitigation Steps
In `VaultImplementation._validateCommitment()` check that `deadline > block.timestamp`.