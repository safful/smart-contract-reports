## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-17

# [Malicious Users Can Drain The Assets Of Vault. (Due to not being ERC4626 Complaint)](https://github.com/code-423n4/2023-01-popcorn-findings/issues/471) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L294


# Vulnerability details

### Impact

Malicious users can drain the assets of the vault.

### POC

The `withdraw` function users `convertToShares` to convert the assets to the amount of shares. These shares are burned from the users account and the assets are returned to the user.

The function `withdraw` is shown below:

```solidity
function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public nonReentrant syncFeeCheckpoint returns (uint256 shares) {
        if (receiver == address(0)) revert InvalidReceiver();

        shares = convertToShares(assets);
/// .... [skipped the code]
```

The function `convertToShares` is shown below:

```solidity
function convertToShares(uint256 assets) public view returns (uint256) {
        uint256 supply = totalSupply(); // Saves an extra SLOAD if totalSupply is non-zero.

        return
            supply == 0
                ? assets
                : assets.mulDiv(supply, totalAssets(), Math.Rounding.Down);
    }
```

It uses `Math.Rounding.Down` , but it should use `Math.Rounding.Up`

Assume that the vault with the following state:

- Total Asset = 1000 WETH
- Total Supply = 10 shares

Assume that Alice wants to withdraw 99 WETH from the vault. Thus, she calls the **`Vault.withdraw(99 WETH)`** function.

The calculation would go like this:

```solidity
assets = 99
return value = assets * supply / totalAssets()
return value = 99 * 10 / 1000
return value = 0
```

The value would be rounded round to zero. This will be the amount of shares burned from users account, which is zero. 

Hence user can drain the assets from the vault without burning their shares. 

> Note : A similar issue also exists in `mint` functionality where `Math.Rounding.Down` is used and `Math.Rounding.Up` should be used.
> 

### Recommendation

Use `Math.Rounding.Up` instead of `Math.Rounding.Down`.

As per OZ implementation here is the rounding method that should be used:

`deposit` : `convertToShares` → `Math.Rounding.Down`

`mint` : `converttoAssets` → `Math.Rounding.Up`

`withdraw` : `convertToShares` → `Math.Rounding.Up`

`redeem` : `convertToAssets` →  `Math.Rounding.Down`