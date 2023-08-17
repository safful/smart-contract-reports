## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-13

# [Processing an epoch must be done in a timely manner, but can be halted by non liquidated expired liens](https://github.com/code-423n4/2023-01-astaria-findings/issues/369) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L303-L305


# Vulnerability details

## Impact
As pointed out in the [Spearbit audit](https://github.com/spearbit-audits/review-astaria/issues/121): 

> If the processEpoch() endpoint does not get called regularly (especially close to the epoch boundaries), the updated currentEpoch would lag behind the actual expected value and this will introduce arithmetic errors in formulas regarding epochs and timestamps.

This can cause a problem because `processEpoch()` cannot be called when there are open liens, and liens may remain open in the event that a lien expires and isn't promptly liquidated.


## Proof of Concept

`processEpoch()` contains the following check to ensure that all liens are closed before the epoch is processed:

```
if (s.epochData[s.currentEpoch].liensOpenForEpoch > 0) {
  revert InvalidState(InvalidStates.LIENS_OPEN_FOR_EPOCH_NOT_ZERO);
}
```

The accounting considers a lien open (via `s.epochData[s.currentEpoch].liensOpenForEpoch`) unless this value is decremented, which happens in three cases: when (a) the full payment is made, (b) the lien is bought out, or (c) the lien is liquidated.

In the event that a lien expires and nobody calls `liquidate()` (for example, if the NFT seems worthless and no other user wants to pay the gas to execute the function for the fee), this would cause `processEpoch()` to fail, and could create delays in the epoch processing and cause the accounting issues pointed out in the previous audit.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Astaria should implement a monitoring solution to ensure that `liquidate()` is always called promptly for expired liens, and that `processEpoch()` is always called promptly when the epoch ends.