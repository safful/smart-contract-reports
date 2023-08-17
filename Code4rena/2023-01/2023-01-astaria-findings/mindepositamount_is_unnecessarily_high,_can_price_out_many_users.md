## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-14

# [minDepositAmount is unnecessarily high, can price out many users](https://github.com/code-423n4/2023-01-astaria-findings/issues/367) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L96-L108


# Vulnerability details

## Impact
The `minDepositAmount()` function is intended to ensure that all depositors into Public Vaults are depositing more than dust to protect against attacks like the 4626 front running attack. It is calculated as follows:

```
  function minDepositAmount()
    public
    view
    virtual
    override(ERC4626Cloned)
    returns (uint256)
  {
    if (ERC20(asset()).decimals() == uint8(18)) {
      return 100 gwei;
    } else {
      return 10**(ERC20(asset()).decimals() - 1);
    }
  }
```
For assets with 18 decimals, this calculation is totally reasonable, as the minimum deposit is just 100 gwei (1/10k of a USD). However, for other assets, the formula returns 0.1 tokens, regardless of value.

While this may be fine for low-priced tokens, it leads to a minimum deposit for tokens like WBTC to be over $2000 USD, certainly too high for many user and not aligned with the intended behavior (which can be seen in the 18 decimal base case).

## Proof of Concept

- WBTC is 8 decimals and has a value of ~$20k USD
- the minimum deposit is calculated as `return 10**(ERC20(asset()).decimals() - 1);`
- for WBTC, this would return 10 ** 7
- `20,000 * (10 ** 7) / (10 ** 8)` = `2000 USD`

This is far out of line with the `1/10,000 USD` expected for tokens with 18 decimals.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Since our requirements for the size of deposit are so low, we can customize a formula that ensures we get a small final result in all cases. For example:

```
if (ERC20(asset()).decimals() < 4) {
    return 10**(ERC20(asset()).decimals() - 1);
else if (ERC20(asset()).decimals() < 8) {
    return 10**(ERC20(asset()).decimals() - 2);
} else {
    return 10**(ERC20(asset()).decimals() - 6);
}
```