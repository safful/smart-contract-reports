## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-01

# [Adversary can abuse a quirk of compound redemption to manipulate the underlying exchange rate and maliciously disable cToken collaterals](https://github.com/code-423n4/2023-01-reserve-findings/issues/310) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/FiatCollateral.sol#L121-L165


# Vulnerability details

## Impact

Adversary can maliciously disable cToken collateral to cause loss to rToken during restructuring

## Proof of Concept

    if (referencePrice < prevReferencePrice) {
        markStatus(CollateralStatus.DISABLED);
    }

CTokenNonFiatCollateral and CTokenFiatCollateral both use the default refresh behavior presented in FiatCollateral which has the above lines which automatically disables the collateral if the reference price ever decreases. This makes the assumption that cToken exchange rates never decrease but this is an incorrect assumption and can be exploited by an attacker to maliciously disable a cToken being used as collateral.

[CToken.sol](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L480-L505)

        uint redeemTokens;
        uint redeemAmount;
        /* If redeemTokensIn > 0: */
        if (redeemTokensIn > 0) {
            /*
             * We calculate the exchange rate and the amount of underlying to be redeemed:
             *  redeemTokens = redeemTokensIn
             *  redeemAmount = redeemTokensIn x exchangeRateCurrent
             */
            redeemTokens = redeemTokensIn;
            redeemAmount = mul_ScalarTruncate(exchangeRate, redeemTokensIn);
        } else {
            /*
             * We get the current exchange rate and calculate the amount to be redeemed:
             *  redeemTokens = redeemAmountIn / exchangeRate
             *  redeemAmount = redeemAmountIn
             */

            // @audit redeemTokens rounds in favor of the user

            redeemTokens = div_(redeemAmountIn, exchangeRate);
            redeemAmount = redeemAmountIn;
        }

The exchange rate can be manipulated by a tiny amount during the redeem process. The focus above is the scenario where the user requests a specific amount of underlying. When calculating the number of cTokens to redeem for a specific amount of underlying it rounds IN FAVOR of the user. This allows the user to redeem more underlying than the exchange rate would otherwise imply. Because the user can redeem *slightly* more than intended they can create a scenario in which the exchange rate actually drops after they redeem. This is because compound calculates the exchange rate dynamically using the current supply of cTokens and the assets under management.

[CToken.sol](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/CToken.sol#L293-L312)

    function exchangeRateStoredInternal() virtual internal view returns (uint) {
        uint _totalSupply = totalSupply;
        if (_totalSupply == 0) {
            /*
             * If there are no tokens minted:
             *  exchangeRate = initialExchangeRate
             */
            return initialExchangeRateMantissa;
        } else {
            /*
             * Otherwise:
             *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
             */
            uint totalCash = getCashPrior();
            uint cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
            uint exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;


            return exchangeRate;
        }
    }

The exchangeRate when _totalSupply != 0 is basically:

    exchangeRate = netAssets * 1e18 / totalSupply

Using this formula for we can now walk through an example of how this can be exploited

Example:

cTokens always start at a whole token ratio of 50:1 so let's assume this ratio to begin with. Let's use values similar to the current supply of cETH which is ~15M cETH and ~300k ETH. We'll start by calculating the current ratio:

    exchangeRate = 300_000 * 1e18 * 1e18 / 15_000_000 * 1e8 = 2e26 

Now to exploit the ratio we request to redeem 99e8 redeemAmount which we can use to calculate the amount of tokens we need to burn:

    redeemAmount = 99e8 * 1e18 / 2e26 = 1.98 -> 1

After truncation the amount burned is only 1. Now we can recalculate our ratio:

    exchangeRate = ((300_000 * 1e18 * 1e18) - 99e8) / ((15_000_000 * 1e8) - 1) = 199999999999999933333333333

The ratio has now been slightly decreased. In CTokenFiatCollateral the exchange rate is truncated to 18 dp so:

    (referencePrice < prevReferencePrice) -> (19999999999999993 <  2e18) == true 

This results in that the collateral is now disabled. This forces the vault to liquidate their holdings to convert to a backup asset. This will almost certainly incur losses to the protocol that were maliciously inflicted.

The path to exploit is relatively straightforward:

`refresh()` cToken collateral to store current rate -> Manipulate compound rate via redemption -> `refresh()` cToken collateral to disable

## Tools Used

Manual Review

## Recommended Mitigation Steps

Since the issue is with the underlying compound contracts, nothing can make the attack impossible but it can be made sufficiently difficult. The simplest deterrent would be to implement a rate error value (i.e. 100) so that the exchange rate has to drop more than that before the token is disabled. The recommended value for this is a bit more complicated to unpack. The amount that the exchange rate changes heavily depends on the number of cTokens minted. The larger the amount the less it changes. Additionally a malicious user can make consecutive redemptions to lower the rate even further. Using an error rate of 1e12 would make it nearly impossible for this to be exploited while still being very sensitive to real (and concerning) changes in exchange rate.

    -   if (referencePrice < prevReferencePrice) {
    +   if (referencePrice < prevReferencePrice - rateError) {
            markStatus(CollateralStatus.DISABLED);
        }