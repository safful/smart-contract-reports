## Tags

- bug
- 3 (High Risk)
- sponsor acknowledged
- upgraded by judge
- satisfactory
- selected for report
- H-04

# [Reserved token rounding can be abused to honeypot and steal user's funds](https://github.com/code-423n4/2022-10-juicebox-findings/issues/191) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721DelegateStore.sol#L1233


# Vulnerability details

## Description

When the project wishes to mint reserved tokens, they call mintReservesFor which allows minting up to the amount calculated by DelegateStore's _numberOfReservedTokensOutstandingFor. The function has this line:

```
// No token minted yet? Round up to 1.
if (_storedTier.initialQuantity == _storedTier.remainingQuantity) return 1;
```

In order to ease calculations, if reserve rate is not 0 and no token has been minted yet, the function allows a single reserve token to be printed. It turns out that this introduces a very significant risk for users. Projects can launch with several tierIDs of similar contribution size, and reserve rate as low as 1%. Once a victim contributes to the project, it can instantly mint a single reserve token of all the rest of the tiers. They can then redeem the reserve token and receive most of the user's contribution, without putting in any money of their own.

Since this attack does not require setting "dangerous" flags like lockReservedTokenChanges or lockManualMintingChanges, it represents a very considerable threat to unsuspecting users. Note that the attack circumvents user voting or any funding cycle changes which leave time for victim to withdraw their funds. 

## Impact

Honeypot project can instantly take most of first user's contribution.

## Proof of Concept

New project launches, with 10 tiers, of contributions 1000, 1050, 1100, ...

Reserve rate is set to 1% and redemption rate = 100%

User contributes 1100 and gets a Tier 3 NFT reward. 

Project immediately mints Tier 1,  Tier 2, Tier 4,... Tier 10 reserve tokens, and redeems all the reserve tokens.

Project's total weight = 12250

Reserve token weight = 11150

Malicious project cashes 1100 (overflow) * 11150 / 12250 = ~1001 tokens.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Don't round up outstanding reserve tokens as it represents too much of a threat.