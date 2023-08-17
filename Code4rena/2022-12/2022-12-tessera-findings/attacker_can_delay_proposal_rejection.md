## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- M-02

# [Attacker can delay proposal rejection](https://github.com/code-423n4/2022-12-tessera-findings/issues/24) 

# Lines of code

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/modules/OptimisticListingSeaport.sol#L145


# Vulnerability details

## Impact

In `OptimisticListingSeaport.rejectProposal`, it revert if `proposedListing.collateral < _amount`. An attacker can therefore monitor the mempool, reducing the `proposedListing.collateral` to `_amount - 1` by frontruning the `rejectProposal` call and delay the rejection. The attacker may even be able to deny the rejection when the deadline passes.

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/modules/OptimisticListingSeaport.sol#L145

```solidity
        if (proposedListing.collateral < _amount) revert InsufficientCollateral();
```

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/modules/OptimisticListingSeaport.sol#L153

```solidity
        proposedListing.collateral -= _amount;
```

## Proof of Concept

1. Attacker propose at 10000 collateral at a very low price
2. Bob try to reject it by purchasing the 10000 collateral
3. Attacker see Bob's tx in the mempool, frontrun it to reject 1 unit
4. The proposedListing.collateral is now 9999
5. Bob's call reverted
6. This keep happening until PROPOSAL_PERIOD pass or Bob gave up because of gas paid on failing tx
7. Attacker buy the NFT at a very low price

## Recommended Mitigation Steps

When `proposedListing.collateral < _amount`, set _amount to proposedListing.collateral and refund the excess.