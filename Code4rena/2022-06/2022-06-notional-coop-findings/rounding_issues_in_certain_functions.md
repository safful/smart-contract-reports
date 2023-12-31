## Tags

- bug
- 3 (High Risk)
- sponsor confirmed
- Notional

# [Rounding Issues In Certain Functions](https://github.com/code-423n4/2022-06-notional-coop-findings/issues/155) 

# Lines of code

https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L52
https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L134


# Vulnerability details

## Background

Per EIP 4626's Security Considerations (https://eips.ethereum.org/EIPS/eip-4626)

> Finally, ERC-4626 Vault implementers should be aware of the need for specific, opposing rounding directions across the different mutable and view methods, as it is considered most secure to favor the Vault itself during calculations over its users:
>
> - If (1) it’s calculating how many shares to issue to a user for a certain amount of the underlying tokens they provide or (2) it’s determining the amount of the underlying tokens to transfer to them for returning a certain amount of shares, it should round *down*.
> - If (1) it’s calculating the amount of shares a user has to supply to receive a given amount of the underlying tokens or (2) it’s calculating the amount of underlying tokens a user has to provide to receive a certain amount of shares, it should round *up*.

Thus, the result of the `previewMint` and `previewWithdraw` should be rounded up.

## Proof-of-Concept

The current implementation of `convertToShares` function will round down the number of shares returned due to how solidity handles Integer Division. ERC4626 expects the returned value of `convertToShares` to be rounded down. Thus, this function behaves as expected.

[https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L52](https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L52)

```solidity
function convertToShares(uint256 assets) public view override returns (uint256 shares) {
    uint256 supply = totalSupply();
    if (supply == 0) {
        // Scales assets by the value of a single unit of fCash
        uint256 unitfCashValue = _getPresentValue(uint256(Constants.INTERNAL_TOKEN_PRECISION));
        return (assets * uint256(Constants.INTERNAL_TOKEN_PRECISION)) / unitfCashValue;
    }

    return (assets * totalSupply()) / totalAssets();
}
```

ERC 4626 expects the result returned from `previewWithdraw` function to be rounded up. However, within the `previewWithdraw` function, it calls the `convertToShares` function. Recall earlier that the `convertToShares` function returned a rounded down value,  thus `previewWithdraw` will return a rounded down value instead of round up value. Thus, this function does not behave as expected.

[https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L134](https://github.com/code-423n4/2022-06-notional-coop/blob/6f8c325f604e2576e2fe257b6b57892ca181509a/notional-wrapped-fcash/contracts/wfCashERC4626.sol#L134)

```solidity
function previewWithdraw(uint256 assets) public view override returns (uint256 shares) {
    if (hasMatured()) {
        shares = convertToShares(assets);
    } else {
        // If withdrawing non-matured assets, we sell them on the market (i.e. borrow)
        (uint16 currencyId, uint40 maturity) = getDecodedID();
        (shares, /* */, /* */) = NotionalV2.getfCashBorrowFromPrincipal(
            currencyId,
            assets,
            maturity,
            0,
            block.timestamp,
            true
        );
    }
}
```

`previewWithdraw` and `previewMint` functions rely on `NotionalV2.getfCashBorrowFromPrincipal` and `NotionalV2.getDepositFromfCashLend` functions. Due to the nature of time-boxed contest, I was unable to verify if `NotionalV2.getfCashBorrowFromPrincipal` and `NotionalV2.getDepositFromfCashLend` functions return a rounded down or up value. If a rounded down value is returned from these functions, `previewWithdraw` and `previewMint` functions would not behave as expected.

## Impact

Other protocols that integrate with Notional's fCash wrapper might wrongly assume that the functions handle rounding as per ERC4626 expectation. Thus, it might cause some intergration problem in the future that can lead to wide range of issues for both parties.

## Recommended Mitigation Steps

Ensure that the rounding of vault's functions behave as expected. Following are the expected rounding direction for each vault function:

- previewMint(uint256 shares) - Round Up ⬆

- previewWithdraw(uint256 assets) - Round Up ⬆

- previewRedeem(uint256 shares) - Round Down ⬇

- previewDeposit(uint256 assets) - Round Down ⬇

- convertToAssets(uint256 shares) - Round Down ⬇

- convertToShares(uint256 assets) - Round Down ⬇

`previewMint` returns the  amount of assets that would be deposited to mint specific amount of shares. Thus, the amount of assets must be rounded up, so that the vault won't be shortchanged.

`previewWithdraw` returns the amount of shares that would be burned to withdraw specific amount of asset. Thus, the amount of shares must to be rounded up, so that the vault won't be shortchanged.

Following is the OpenZeppelin's vault implementation for rounding reference:

[https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20TokenizedVault.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20TokenizedVault.sol)

Alternatively, if such alignment of rounding could not be achieved due to technical limitation, at the minimum, document this limitation in the comment so that the developer performing the integration is aware of this.

