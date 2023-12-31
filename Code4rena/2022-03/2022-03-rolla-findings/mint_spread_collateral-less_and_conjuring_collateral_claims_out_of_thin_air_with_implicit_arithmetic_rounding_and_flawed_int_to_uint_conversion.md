## Tags

- bug
- 3 (High Risk)
- resolved
- sponsor confirmed

# [Mint spread collateral-less and conjuring collateral claims out of thin air with implicit arithmetic rounding and flawed int to uint conversion](https://github.com/code-423n4/2022-03-rolla-findings/issues/31) 

# Lines of code

https://github.com/code-423n4/2022-03-rolla/blob/main/quant-protocol/contracts/libraries/QuantMath.sol#L137
https://github.com/code-423n4/2022-03-rolla/blob/main/quant-protocol/contracts/libraries/QuantMath.sol#L151
https://github.com/code-423n4/2022-03-rolla/blob/main/quant-protocol/contracts/libraries/SignedConverter.sol#L28


# Vulnerability details

## Impact

This report presents 2 different incorrect behaviour that can affect the correctness of math calculations
1. Unattended Implicit rounding in QuantMath.sol `div` and `mul`
2. Inappropriate method of casting integer to unsigned integer in SignedConverter.sol `intToUint`

Bug 1 affects the correctness when calculating collateral required for `_mintSpread`. Bug 2 expands the attack surface and allows attackers to target the `_claimCollateral` phase instead. Both attacks may result in tokens being stolen from Controller in the worst case, but is most likely too costly to exploit under current BNB chain environment. The potential impact however, should not be taken lightly, since it is known that the ethereum environment in highly volatile and minor changes in the environment can suddenly make those bugs cheap to exploit.


## Proof of Concept

In this section, we will first present bug 1, and then demonstrate how this bug can be exploited. Then we will discuss how bug 2 opens up more attack chances and go over another PoC.

Before getting started, we should go over an important concept while dealing with fixed point number -- rounding.
Math has no limits on precision, but computers do. This problem is especially critical to systems handling large amount of "money" that is allowed to be arbitrarily divided. A common way for ethereum smart contract developers to handle this is through rounding numbers. Rolla is no exception.

In QuantMath, Rolla explicitly wrote the `toScaledUint` function to differentiate between rounding numbers up or down when scaling numbers to different precision (or we call it `_decimals` here). The intended usage is to scale calculated numbers (amount of tokens) up when Controller is the receiver, and scale it down when Controller is sender. In theory, this function should guarantee Controller can never "lose tokens" due to rounding.

```
library QuantMath {
    ...
    struct FixedPointInt {
        int256 value;
    }

    int256 private constant _SCALING_FACTOR = 1e27;
    uint256 private constant _BASE_DECIMALS = 27;

    ...

    function toScaledUint(
        FixedPointInt memory _a,
        uint256 _decimals,
        bool _roundDown
    ) internal pure returns (uint256) {
        uint256 scaledUint;

        if (_decimals == _BASE_DECIMALS) {
            scaledUint = _a.value.intToUint();
        } else if (_decimals > _BASE_DECIMALS) {
            uint256 exp = _decimals - _BASE_DECIMALS;
            scaledUint = (_a.value).intToUint() * 10**exp;
        } else {
            uint256 exp = _BASE_DECIMALS - _decimals;
            uint256 tailing;
            if (!_roundDown) {
                uint256 remainer = (_a.value).intToUint() % 10**exp;
                if (remainer > 0) tailing = 1;
            }
            scaledUint = (_a.value).intToUint() / 10**exp + tailing;
        }

        return scaledUint;
    }
    ...
}
```

In practice, the above function also works quite well (sadly, not perfect, notice the `intToUint` function within. We will come back to this later), but it only works if we can promise that before entering this function, all numbers retain full precision and is not already rounded. This is where `div` and `mul` comes into play. As we can easily see in the snippet below, both functions involve the division operator '/', which by default discards the decimal part of the calculated result (be aware to not confuse this with the `_decimal` used while scaling FixedPointInt). The operation here results in an implicit round down, which limits the effectiveness of  explicit rounding in `toScaledUint` showned above.

```
    function mul(FixedPointInt memory a, FixedPointInt memory b)
        internal
        pure
        returns (FixedPointInt memory)
    {
        return FixedPointInt((a.value * b.value) / _SCALING_FACTOR);
    }


    function div(FixedPointInt memory a, FixedPointInt memory b)
        internal
        pure
        returns (FixedPointInt memory)
    {
        return FixedPointInt((a.value * _SCALING_FACTOR) / b.value);
    }
```

Now let's see how this implicit rounding can causes troubles. We start with the `_mintSpread` procedure creating a call credit spread. For brevity, the related code is not shown, but here's a summary of what is done.

* `Controller._mintSpread`
  * `QuantCalculator.getCollateralRequirement`
    * `FundsCalculator.getCollateralRequirement`
      * `FundsCalculator.getOptionCollateralRequirement`
        * `FundsCalculator.getCallCollateralRequirement`
          * scales `_qTokenToMintStrikePrice` from
             `_strikeAssetDecimals (8)` to `_BASE_DECIMALS (27)`
          * scales `_qTokenForCollateralStrikePrice` from
             `_strikeAssetDecimals (8)` to `_BASE_DECIMALS (27)`
          * `collateralPerOption = (collateralStrikePrice.sub(mintStrikePrice)).div(collateralStrikePrice)`
        * scale `_optionsAmount` from `_optionsDecimals (18)` to `_BASE_DECIMALS (27)`
        * `collateralAmount = _optionsAmount.mul(collateralPerOption)`
      * uses `qTokenToMint.underlyingAsset` (weth or wbtc) as collateral
    * scale and round up `collateralAmountFP` from `_BASE_DECIMALS (27)` to `payoutDecimals (18)`

If we extract all the math related stuff, it would be something like below

```
def callCreditSpreadCollateralRequirement(_qTokenToMintStrikePrice, _qTokenForCollateralStrikePrice, _optionsAmount):
        X1 = _qTokenToMintStrikePrice * 10^19
        X2 = _qTokenForCollateralStrikePrice * 10^19
        X3 = _optionsAmount * 10^9

        assert X1 < X2          #credit spread

        Y1 = (X2 - X1) * 10^27 // X2    #implicit round down due to div
        Y2 = Y1 * X3 // 10^27   #implicit round down due to mul

        Z = Y2 // 10^9
        if Y2 % 10^9 > 0:       #round up since we are minting spread (Controller is receiver)
                Z+=1
        return Z
```

Both implicit round downs can be abused, but we shall focus on the `mul` one here.
Assume we follow the following actions

1. create option `A` with strike price `10 + 10^-8 BUSD (10^9 + 1 under 8 decimals) <-> 1 WETH`
2. create option `B` with strike price `10 BUSD (10^9 under 8 decimals) <-> 1 WETH`
3. mint `10^-18` (1 under 18 decimals) option `A`
        3-1. `pay 1 eth`
4. mint `10^-18` (1 under 18 decimals) spread `B` with `A` as collateral
        4-1. `X1 = _qTokenToMintStrikePrice * 10^19 = 10^9 * 10^19 = 10^28`
        4-2. `X2 = _qTokenToMintStrikePrice * 10^19 = (10^9 + 1) * 10^19 = 10^28 + 10^19`
        4-3. `X3 = _optionsAmount * 10^9 = 1 * 10^9 = 10^9`
        4-4. `Y1 = (X2 - X1) * 10^27 // X2 = (10^28 + 10^19 - 10^28) * 10^27 // (10^28 + 10^19) = 99999999000000000`
        4-5. `Y2 = Y1 * X3 // 10^27 = 99999999000000000 * 10^9 / 10^27 = 0`
        4-6. `Z = Y2 // 10^9 = 0`
        4-7. `Y2 % 10^9 = 0` so `Z` remains unchanged

We minted a call credit spread without paying any fee.

Now let's think about how to extract the value we conjured out of thin air. To be able to withdraw excessive collateral, we can choose to do a excercise+claim or neutralize current options. Here we take the neutralize path.

For neutralizing spreads, the procedure is basically the same as minting spreads, except that the explicit round down is taken since `Controller` is the payer here. The neutralize procedure returns the `qToken` used as collateral and pays the collateral fee back. The math part can be summarized as below.

```
def neutralizeCreditSpreadCollateralRequirement(_qTokenToMintStrikePrice, _qTokenForCollateralStrikePrice, _optionsAmount):
        X1 = _qTokenToMintStrikePrice * 10^19
        X2 = _qTokenForCollateralStrikePrice * 10^19
        X3 = _optionsAmount * 10^9

        assert X1 < X2          #credit spread

        Y1 = (X2 - X1) * 10^27 // X2    #implicit round down due to div
        Y2 = Y1 * X3 // 10^27   #implicit round down due to mul

        Z = Y2 // 10^9  #explicit scaling
        return Z
```

There are two challenges that need to be bypassed, the first one is to avoid implicit round down in `mul`, and the second is to ensure the revenue is not rounded away during explicit scaling.
To achieve this, we first mint `10^-9 + 2 * 10^-18` spreads seperately (10^9 + 2 under 18 decimals), and as shown before, no additional fees are required while minting spread from original option.
Then we neutralize all those spreads at once, the calculation is shown below

1. neutralize `10^-9 + 2 * 10^-18` (10^9 + 2 under 18 decimals) spread `B`
        4-1. `X1 = _qTokenToMintStrikePrice * 10^19 = 10^9 * 10^19 = 10^28`
        4-2. `X2 = _qTokenToMintStrikePrice * 10^19 = (10^9 + 1) * 10^19 = 10^28 + 10^19`
        4-3. `X3 = _optionsAmount * 10^9 = (10^9 + 2) * 10^9 = 10^18 + 2`
        4-4. `Y1 = (X2 - X1) * 10^27 // X2 = (10^28 + 10^19 - 10^28) * 10^27 // (10^28 + 10^19) = 99999999000000000`
        4-5. `Y2 = Y1 * X3 // 10^27 = 99999999000000000 * (10^18 + 2) / 10^27 = 1000000000`
        4-6. `Z = Y2 // 10^9 = 10^9 // 10^9 = 1`

And with this, we managed to generate 10^-18 weth of revenue.

This approach is pretty impractical due to the requirement of minting 10^-18 for `10^9 + 2` times. This montrous count mostly likely requires a lot of gas to pull off, and offsets the marginal revenue generated through our attack. This leads us to explore other possible methods to bypass this limitation.

It's time to start looking at the second bug.

Recall we mentioned the second bug is in `intToUint`, so here's the implementation of it. It is not hard to see that this is actually an `abs` function named as `intToUint`.

```
    function intToUint(int256 a) internal pure returns (uint256) {
        if (a < 0) {
            return uint256(-a);
        } else {
            return uint256(a);
        }
    }
```

Where is this function used? And yes, you guessed it, in `QuantCalculator.calculateClaimableCollateral`. The process of claiming collateral is quite complex, but we will only look at the specific case relevant to the exploit. Before reading code, let's first show the desired scenario. Note that while we wait for expiry, there are no need to sell any option/spread.

1. mint a `qTokenLong` option
2. mint a `qTokenShort` spread with `qTokenLong` as collateral
3. wait until expire, and expect expiryPrice to be between qTokenLong and qTokenShort

```
----------- qTokenLong strike price

----------- expiryPrice

----------- qTokenShort strike price
```

Here is the outline of the long waited claimCollateral for spread.

* `Controller._claimCollateral`
  * `QuantCalculator.calculateClaimableCollateral`
    * `FundsCalculator.getSettlementPriceWithDecimals`
    * `FundsCalculator.getPayout` for qTokenLong
      * qTokenLong strike price is above expiry price, worth 0
    * `FundsCalculator.getCollateralRequirement`
      * This part we saw earlier, omit details
    * `FundsCalculator.getPayout` for qTokenShort
      * uses `qTokenToMint.underlyingAsset` (weth or wbtc) as collateral
      * `FundsCalculator.getPayoutAmount` for qTokenShort
        * scale `_strikePrice` from
          `_strikeAssetDecimals (8)` to `_BASE_DECIMALS (27)`
        * scale `_expiryPrice.price` from
          `_expiryPrice.decimals (8)` to `_BASE_DECIMALS (27)`
        * scale `_amount` from
          `_optionsDecimals (18)` to `_BASE_DECIMALS (27)`
        * `FundsCalculator.getPayoutForCall` for qTokenShort
          * `payoutAmount = expiryPrice.sub(strikePrice).mul(amount).div(expiryPrice)`
    * `returnableCollateral = payoutFromLong.add(collateralRequirement).sub(payoutFromShort)`
    * scale and round down `abs(returnableCollateral)` from `_BASE_DECIMALS (27)` to `payoutDecimals (18)`


Again, we summarize the math part into a function

```
def claimableCollateralCallCreditSpreadExpiryInbetween(_qTokenShortStrikePrice, _qTokenLongStrikePrice, _expiryPrice, _amount):

        def callCreditSpreadCollateralRequirement(_qTokenToMintStrikePrice, _qTokenForCollateralStrikePrice, _optionsAmount):
                X1 = _qTokenToMintStrikePrice * 10^19
                X2 = _qTokenForCollateralStrikePrice * 10^19
                X3 = _optionsAmount * 10^9

                Y1 = (X2 - X1) * 10^27 // X2
                Y2 = Y1 * X3 // 10^27
                return Y2

        def callCreditSpreadQTokenShortPayout(_strikePrice, _expiryPrice, _amount):
                X1 = _strikePrice * 10^19
                X2 = _expiryPrice * 10^19
                X3 = _amount * 10^9

                Y1 = (X2-X1) * X3 // 10^27
                Y2 = Y1 * 10^27 // X2
                return Y2


        assert _qTokenShortStrikePrice > _expiryPrice > _qTokenLongStrikePrice

        A1 = payoutFromLong = 0
        A2 = collateralRequirement = callCreditSpreadCollateralRequirement(_qTokenShortStrikePrice, _qTokenLongStrikePrice, _amount)
        A3 = payoutFromShort = callCreditSpreadQTokenShortPayout(_qTokenShortStrikePrice, _expiryPrice, _amount)

        B1 = A1 + A2 - A3

        Z = abs(B1) // 10^9
        return Z
```

Given the context, it should be pretty easy to imagine what I am aiming here, to make `B1 < 0`. We already know `A1 = 0`, so the gaol basically boils down to making `A2 < A3`. Let's further simplify this requirement and see if the equation is solvable.

```
X = _qTokenLongStrikePrice (8 decimals)
Y = _expiryPrice (8 decimals)
Z = _qTokenShortStrikePrice (8 decimals)
A = _amount (scaled to 27 decimals)

assert X>Y>Z>0
assert X,Y,Z are integers
assert (((X - Z) * 10^27 // X) * A // 10^27) < (((Y - Z) * A // 10^27) * 10^27 // Y)
```

Notice apart from the use of `X` and `Y`, the two sides of the equation only differs by when `A` is mixed into the equation, meaning that if we temporarily ignore the limitation and set `X = Y`, as long as left hand side of equation does an implicit rounding after dividing by X, right hand side will most likely be larger.

Utilizing this, we turn to solve the equation of

```
(X-Z) / X - (Y-Z) / Y < 10^-27
=> Z / Y - Z / X < 10^-27
=> (Z = 1 yields best solution)
=> 1 / Y - 1 / X < 10^-27
=> X - Y < X * Y * 10^-27
=> 0 < X * Y - 10^27 * X + 10^27 * Y

=> require X > Y, so model Y as X - B, where B > 0 and B is an integer
=> 0 < X^2 - B * X - 10^27 * B
```

It is not easy to see that the larger `X` is, the larger the range of allowed `B`. This is pretty important since `B` stands for the range of expiry prices where attack could work, so the larger it is, the less accurate our guess can be to profit.

Apart form range of `B`, value of `X` is the long strike price and upper bound of range `B`, so we would also care about it, a simple estimation shows that `X` must be above `10^13.5 (8 decimals)` for there to be a solution, which amounts to about `316228 BUSD <-> 1 WETH`. This is an extremely high price, but not high enough to be concluded as unreachable in the near future. So let's take a slightly generous number of `10^14 - 1` as X and calculate the revenue generated following this exploit path.

```
0 < (10^14 - 1)^2 - B * (10^14 - 1) - 10^27 * B
=> (10^14 - 1)^2 / (10^14 - 1 + 10^27) > B
=> B <= 9
```

Now we've got the range of profitable expiry price. As we concluded earlier, the range is extremely small with a modest long strike price, but let's settle with this for now and see how much profit can be generated if we get lucky. To calculate profit, we take `_qTokenLongStrikePrice = 10^14 - 1 (8 decimals)`, `_qTokenShortStrikePrice = 1 (8 decimals)`, `_expiryPrice = 10^14 - 2 (8 decimals)` and `_amount = 10^28 (18 decimals)` and plug it back into the function.
1. in `callCreditSpreadCollateralRequirement`
        1-1. `X1 = _qTokenForCollateralStrikePrice * 10^19 = 1 * 10^19 = 10^19`
        1-2. `X2 = _qTokenToMintStrikePrice * 10^19 = (10^14 - 1) * 10^19 = 10^33 - 10^19`
        1-3. `X3 = _optionsAmount * 10^9 = 10^28 * 10^9 = 10^37`
        1-4. `Y1 = (X2 - X1) * 10^27 // X2 = (10^33 - 2 * 10^19) * 10^27 // (10^33 - 10^19) = 999999999999989999999999999`
        1-5. `Y2 = Y1 * X3 // 10^27 = 999999999999989999999999999 * 10^37 // 10^27 = 999999999999989999999999999 * 10^10`
2. in `callCreditSpreadQTokenShortPayout`
        2-1. `X1 = _strikePrice * 10^19 = 1 * 10^19 = 10^19`
        2-2. `X2 = _expiryPrice * 10^19 = (10^14 - 2) * 10^19 = 10^33 - 2 * 10^19`
        2-3. `X3 = _amount * 10^9 = 10^28 * 10^9 = 10^37`
        2-4. `Y1 = (X2 - X1) * X3 // 10^27 = (10^33 - 3 * 10^19) * 10^37 // 10^27 = 99999999999997 * 10^29`
        2-5. `Y2 = Y1 * 10^27 / X2 = (99999999999997 * 10^28) * 10^27 / (10^33 - 2 * 10^19) = 9999999999999899999999999997999999999
3. combine terms
        3-1. `B1 = A1 + A2 - A3 = 0 + 9999999999999899999999999990000000000 - 9999999999999899999999999997999999999 = -2000000001
        3-2. `Z = abs(B1) // 10^9 = 2000000000 // 10^9 = 2

And with this, we managed to squeeze 2 wei from a presumably worthless collateral.

This attack still suffers from several problems
1. cost of WETH in BUSD is way higher than current market
2. need to predict target price accurately to profit
3. requires large amount of WETH to profit

While it is still pretty hard to pull off attack, the requirements seems pretty more likely to be achievable compared to the first version of exploit. Apart from this, there is also the nice property that this attack allows profit to scale with money invested.

This concludes our demonstration of two attacks against the potential flaws in number handling.

## Tools Used

vim, ganache-cli

## Recommended Mitigation Steps

For `div` and `mul`, adding in a similar opt-out round up argument would work. This would require some refactoring of code, but is the only way to fundamentally solve the problem.

For `intToUint`, I still can't understand what the original motive is to design it as `abs` in disguise. Since nowhere in this project would we benefit from the current `abs` behaviour, in my opinion, it would be best to adopt a similar strategy to the `uintToInt` function. If the value goes out of directly convertable range ( < 0), revert and throw an error message.

