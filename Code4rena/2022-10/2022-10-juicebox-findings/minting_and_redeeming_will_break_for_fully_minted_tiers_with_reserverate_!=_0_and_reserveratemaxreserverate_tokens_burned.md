## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- satisfactory
- edited-by-warden
- selected for report
- H-02

# [Minting and redeeming will break for fully minted tiers with reserveRate != 0 and reserveRate/MaxReserveRate tokens burned](https://github.com/code-423n4/2022-10-juicebox-findings/issues/113) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/f9893b1497098241dd3a664956d8016ff0d0efd0/contracts/JBTiered721DelegateStore.sol#L1224-L1259


# Vulnerability details

## Impact

Minting and redeeming become impossible

## Proof of Concept

    uint256 _numberOfNonReservesMinted = _storedTier.initialQuantity -
      _storedTier.remainingQuantity -
      _reserveTokensMinted;

    uint256 _numerator = uint256(_numberOfNonReservesMinted * _storedTier.reservedRate);

    uint256 _numberReservedTokensMintable = _numerator / JBConstants.MAX_RESERVED_RATE;

    if (_numerator - JBConstants.MAX_RESERVED_RATE * _numberReservedTokensMintable > 0)
      ++_numberReservedTokensMintable;

    return _numberReservedTokensMintable - _reserveTokensMinted;

The lines above are taken from JBTiered721DelegateStore#_numberOfReservedTokensOutstandingFor and used to calculate and return the available number of reserve tokens that can be minted. Since the return statement doesn't check that _numberReservedTokensMintable >= _reserveTokensMinted, it will revert under those circumstances. The issue is that there are legitimate circumstances in which this becomes false. If a tier is fully minted then all reserve tokens are mintable. When the tier begins to redeem, _numberReservedTokensMintable will fall under _reserveTokensMinted, permanently breaking minting and redeeming. Minting is broken because all mint functions directly call _numberOfReservedTokensOutstandingFor. Redeeming is broken because the redeem callback (JB721Delegate#redeemParams) calls _totalRedemtionWeight which calls _numberOfReservedTokensOutstandingFor. 

Example:

A tier has a reserveRate of 100 (1/100 tokens reserved) and an initialQuantity of 10000. We assume that the tier has been fully minted, that is, _reserveTokensMinted is 100 and remainingQuantity = 0. Now we begin burning the tokens. Let's run through the lines above after 100 tokens have been burned (remainingQuantity = 100):

_numberOfNonReservedMinted = 10000 - 100 - 100 = 9800

_numerator = 9800 * 100 = 980000

_numberReservedTokensMintable = 980000 / 10000 = 98

Since _numberReservedTokensMintable < _reserveTokensMinted the line will underflow and revert.

JBTiered721DelegateStore#_numberOfReservedTokensOutstandingFor will now revert every time it is called. This affects all minting functions as well as totalRedemptionWeight. Since those functions now revert when called, it is impossible to mint or redeem anymore NFTs.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add a check before returning:

    +   if (_reserveTokensMinted > _numberReservedTokensMintable) {
    +       return 0;
    +   }

        return _numberReservedTokensMintable - _reserveTokensMinted;