## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-12

# [No slippage protection on `stake()` in SafEth.sol](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/150) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L63-L101


# Vulnerability details

## Impact
`mintAmount` is determined both by `totalStakeValueEth` and `preDepositPrice`. While the former is associated with external interactions beyond users' control, the latter should be linked to a slippage control to incentivize more staker participations.  

## Proof of Concept
As can be seen from the code block below, `ethPerDerivative()` serves to get the price of each derivative in terms of ETH. Although it is presumed the prices entailed would be closely/stably pegged 1:1, no one could guarantee the degree of volatility just as what has recently happened to the USDC depeg.

[File: SafEth.sol#L71-L81](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L71-L81)

```solidity
        for (uint i = 0; i < derivativeCount; i++)
            underlyingValue +=
                (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
                    derivatives[i].balance()) /
                10 ** 18;

        uint256 totalSupply = totalSupply();
        uint256 preDepositPrice; // Price of safETH in regards to ETH
        if (totalSupply == 0)
            preDepositPrice = 10 ** 18; // initializes with a price of 1
        else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```
When `underlyingValue` is less than `totalSupply`, `preDepositPrice` will be smaller and inversely make `mintAmount` bigger, and vice versa.

Any slight change in price movement in the same direction can be consistently cumulative and reflective in stake calculations. This can make two stakers calling `stake()` with the same ETH amount minutes apart getting minted different amount of stake ERC20 tokens. 

## Tools Used
Manual

## Recommended Mitigation Steps
Consider having a user inputtable `minMintAmountOut` added in the function parameters of `stake()` and the function logic refactored as follows:

```diff
-    function stake() external payable {
+    function stake(uint256 minMintAmountOut) external payable {

        [... Snipped ...]

        _mint(msg.sender, mintAmount);
+        require(shares >= minSharesOut, "mint amount too low");

        [... Snipped ...]
```
Ideally, this slippage calculation should be featured in the UI, with optionally selectable stake amount impact, e.g. 0.1%.  