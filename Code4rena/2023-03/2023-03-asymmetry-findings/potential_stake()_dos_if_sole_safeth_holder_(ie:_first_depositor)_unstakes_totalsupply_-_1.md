## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-03

# [potential stake() DoS if sole safETH holder (ie: first depositor) unstakes totalSupply - 1](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1016) 

# Lines of code

https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L68-L81
https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L98


# Vulnerability details

## Impact
Potential inability to stake (ie: DoS) if sole safETH user (ie: this would also make them the sole safETH holder) unstakes `totalSupply - 1`.

## Proof of Concept
The goal of this POC is to prove that this line can revert https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L98
```
        uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
```
This can occur if the attacker can cause `preDepositPrice = 0`.

A user who is the first staker will be the sole holder of 100% of `totalSupply` of safETH.
They can then unstake (and therefore burn) `totalSupply - 1` leaving a total of 1 wei of safETH in circulation.

In earlier lines in `stake()` https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L77-L81, we see 
```
        uint256 totalSupply = totalSupply();
        uint256 preDepositPrice; // Price of safETH in regards to ETH
        if (totalSupply == 0)
            preDepositPrice = 10 ** 18; // initializes with a price of 1
        else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```
With `totalSupply = 1`, we see that the above code block will execute the `else` code path, and that if `underlyingValue = 0`, then `preDepositPrice = 0`.

`underlyingValue` is set in earlier lines: https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L68-L75
```
        uint256 underlyingValue = 0;

        // Getting underlying value in terms of ETH for each derivative
        for (uint i = 0; i < derivativeCount; i++)
            underlyingValue +=
                (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
                    derivatives[i].balance()) /
                10 ** 18;
```

For a simple case, assume there is 1 derivative with 100% weight. Let's use rETH for this example since the derivative can get its `ethPerDerivative` price from an AMM. In this case:
- Assume the `ethPerDerivative()` value has been manipulated in the underlying AMM pool such that 1 derivative ETH is worth less than 1 ETH. eg: 1 rETH = 9.99...9e17 ETH
- In this case, also assume that since there is 1 wei of safETH circulating, there should be 1 wei of ETH staked through the protocol, and therefore `derivatives[i].balance() = 1 wei`.

This case will result in `underlyingValue += (9.99...9e17 * 1) / 10 ** 18 = 0`.

We can see that it is therefore possible to cause a divide by 0 revert and malfunction of the `stake()` function.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Assuming the deployment process will set up at least 1 derivative with a weight, simply adding a `stake()`  operation of 0.5 ETH as the first depositor as part of the deployment process avoids the case where safETH totalSupply drops to 1 wei.
Otherwise, within `unstake()` it is also possible to require that `totalSupply` does not fall between 0 and `minimumSupply` where `minimumSupply` is, for example, the configured `minAmount`.