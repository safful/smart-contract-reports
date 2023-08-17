## Tags

- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- MR-NEW

# [Rounding loss in and with approxPrice()](https://github.com/code-423n4/2023-05-asymmetry-mitigation-findings/issues/71) 

# Rounding loss in and with approxPrice()

https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L87-L119
https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L359-L373

## Description
[`SafEth.approxPrice()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L359-L373) contains a rounding loss of the form `a/k + b/k <= (a + b)/k` which can be refactored as follows:
```diff
for (uint256 i = 0; i < count; i++) {
    if (!derivatives[i].enabled) continue;
    IDerivative derivative = derivatives[i].derivative;
    underlyingValue +=
-         (derivative.ethPerDerivative() * derivative.balance()) /
-         1e18;
+         (derivative.ethPerDerivative() * derivative.balance())
}
if (safEthTotalSupply == 0 || underlyingValue == 0) return 1e18;
- return (1e18 * underlyingValue) / safEthTotalSupply;
+ return underlyingValue / safEthTotalSupply;
```

But even with this refactoring, in `stake()` we have [the line](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L114)
`mintedAmount = (totalStakeValueEth * 1e18) / preDepositPrice;`
where `preDepositPrice = approxPrice()`, so this suffers a rounding loss of the form `a/(b/c) >= a*c/b`.
We would want to refactor this line to
`mintedAmount = (totalStakeValueEth * 1e18 * safEthTotalSupply) / underlyingValue;`.

We have another [case of `a/k + b/k <= (a + b)/k` in `stake()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L98-L112):
```solidity
for (uint256 i = 0; i < count; i++) {
    ...
    uint256 derivativeReceivedEthValue = (derivative
        .ethPerDerivative() * depositAmount) / 1e18;
    totalStakeValueEth += derivativeReceivedEthValue;
    ...
}
```
So we can do the same here and defer the division by `1e18` to after the summation, which gives us
```diff
- mintedAmount = (totalStakeValueEth * 1e18 * safEthTotalSupply) / underlyingValue;
+ mintedAmount = (totalStakeValueEth * safEthTotalSupply) / underlyingValue;
```

## Recommendation
Do the above refactoring in `approxPrice()`. This function is still needed to estimate `_minOut`.
Note that `approxPrice()` is calculated anew in the [emitted event at the end of `stake()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/SafEth.sol#L118). This means that this was not the price paid for the stake just made, but the price to pay for the next stake. It seems more appropriate to emit the price just paid, and this would also save gas by just reusing the already calculated price.

As for `stake()`, the rounding loss would have to be eliminated by inlining the calculation. Note that the two for-loops may also be combined. Here is a complete refactoring:
```solidity
function stake(
    uint256 _minOut
) external payable nonReentrant returns (uint256 mintedAmount) {
    require(!pauseStaking, "staking is paused");
    require(msg.value >= minAmount, "amount too low");
    require(msg.value <= maxAmount, "amount too high");
    require(totalWeight > 0, "total weight is zero");

    uint256 count = derivativeCount;
    uint256 underlyingValue = 0;
    uint256 totalStakeValueEth = 0; // total amount of derivatives staked by user in eth
    for (uint256 i = 0; i < count; i++) {
        IDerivative derivative = derivatives[i].derivative;
        if (!derivative.enabled) continue;

        underlyingValue += (derivative.ethPerDerivative() * derivative.balance()); // 36 decimals

        uint256 weight = derivatives[i].weight;
        if (weight == 0) continue;

        uint256 ethAmount = (msg.value * weight) / totalWeight;
        if (ethAmount > 0) {
            // This is slightly less than ethAmount because slippage
            uint256 depositAmount = derivative.deposit{value: ethAmount}();
            uint256 derivativeReceivedEthValue = (derivative
                .ethPerDerivative() * depositAmount);
            totalStakeValueEth += derivativeReceivedEthValue; // 36 decimals
        }
    }
    // mintedAmount represents a percentage of the total assets in the system
    uint256 safEthTotalSupply = totalSupply();
    mintedAmount = (safEthTotalSupply == 0 || underlyingValue == 0)
        ? totalStakeValueEth / 10 ** 18
        : totalStakeValueEth * safEthTotalSupply / underlyingValue;
    require(mintedAmount > _minOut, "mint amount less than minOut");

    _mint(msg.sender, mintedAmount);
    emit Staked(msg.sender, msg.value, totalStakeValueEth, approxPrice());
}
```
where, following the discussion above, the last part may be replaced with
```solidity
uint256 preDepositPrice;
uint256 safEthTotalSupply = totalSupply();
if (safEthTotalSupply == 0 || underlyingValue == 0) {
    preDepositPrice = 1e18;
    mintedAmount = totalStakeValueEth / 10 ** 18;
} else {
    preDepositPrice = (underlyingValue / safEthTotalSupply) / 1e18;
    mintedAmount = totalStakeValueEth * safEthTotalSupply / underlyingValue;
}
require(mintedAmount > _minOut, "mint amount less than minOut");

_mint(msg.sender, mintedAmount);
emit Staked(msg.sender, msg.value, totalStakeValueEth, preDepositPrice);
```