## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- H-03

# [Malicious Users Can Drain The Assets Of Auto Compound Vault](https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/178) 

# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/PirexERC4626.sol#L156
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L199
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L315


# Vulnerability details

## Proof of Concept

> Note: This issue affects both the AutoPxGmx and AutoPxGlp vaults. Since the root cause is the same, the PoC of AutoPxGlp vault is omitted for brevity.

The `PirexERC4626.convertToShares` function relies on the `mulDivDown` function in Line 164 when calculating the number of shares needed in exchange for a certain number of assets. Note that the computation is rounded down, therefore, if the result is less than 1 (e.g. 0.9), Solidity will round them down to zero. Thus, it is possible that this function will return zero.

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/PirexERC4626.sol#L156

```solidity
File: PirexERC4626.sol
156:     function convertToShares(uint256 assets)
157:         public
158:         view
159:         virtual
160:         returns (uint256)
161:     {
162:         uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.
163: 
164:         return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
165:     }
```

The `AutoPxGmx.previewWithdraw` function relies on the `PirexERC4626.convertToShares` function in Line 206. Thus, this function will also "round down".

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L199

```solidity
File: AutoPxGmx.sol
199:     function previewWithdraw(uint256 assets)
200:         public
201:         view
202:         override
203:         returns (uint256)
204:     {
205:         // Calculate shares based on the specified assets' proportion of the pool
206:         uint256 shares = convertToShares(assets);
207: 
208:         // Save 1 SLOAD
209:         uint256 _totalSupply = totalSupply;
210: 
211:         // Factor in additional shares to fulfill withdrawal if user is not the last to withdraw
212:         return
213:             (_totalSupply == 0 || _totalSupply - shares == 0)
214:                 ? shares
215:                 : (shares * FEE_DENOMINATOR) /
216:                     (FEE_DENOMINATOR - withdrawalPenalty);
217:     }
```

The `AutoPxGmx.withdraw` function relies on the `AutoPxGmx.previewWithdraw` function. In certain conditions, the `AutoPxGmx.previewWithdraw` function in Line 323 will return zero if the withdrawal amount causes the division within the `PirexERC4626.convertToShares` function to round down to zero (usually due to a small amount of withdrawal amount).

If the `AutoPxGmx.previewWithdraw` function in Line 323 returns zero, no shares will be burned at Line 332. Subsequently, in Line 336, the contract will transfer the assets to the users. As a result, the users receive the assets without burning any of their shares, effectively allowing them to receive assets for free.

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L315

```solidity
File: AutoPxGmx.sol
315:     function withdraw(
316:         uint256 assets,
317:         address receiver,
318:         address owner
319:     ) public override returns (uint256 shares) {
320:         // Compound rewards and ensure they are properly accounted for prior to withdrawal calculation
321:         compound(poolFee, 1, 0, true);
322:         
323:         shares = previewWithdraw(assets); // No need to check for rounding error, previewWithdraw rounds up.
324: 
325:         if (msg.sender != owner) {
326:             uint256 allowed = allowance[owner][msg.sender]; // Saves gas for limited approvals.
327: 
328:             if (allowed != type(uint256).max)
329:                 allowance[owner][msg.sender] = allowed - shares;
330:         }
331: 
332:         _burn(owner, shares);
333: 
334:         emit Withdraw(msg.sender, receiver, owner, assets, shares);
335: 
336:         asset.safeTransfer(receiver, assets);
337:     }
```

Assume that the vault with the following state:

- Total Asset = 1000 WETH
- Total Supply = 10 shares

Assume that Alice wants to withdraw 99 WETH from the vault. Thus, she calls the `AutoPxGmx.withdraw(99 WETH)` function.

The `PirexERC4626.convertToShares` function will compute the number of shares that Alice needs to burn in exchange for 99 WETH.

```solidity
assets.mulDivDown(supply, totalAssets())
99WETH.mulDivDown(10 shares, 1000WETH)
(99 * 10) / 1000
990 / 1000 = 0.99 = 0
```

However, since Solidity round `0.99` down to `0`, Alice does not need to burn a single share. She will receive 99 WETH for free.

## Impact

Malicious users can withdraw the assets from the vault for free, effectively allowing them to drain the assets of the vault.

## Recommended Mitigation Steps

Ensure that at least 1 share is burned when the users withdraw their assets.

This can be mitigated by updating the `previewWithdraw` function to round up instead of round down when computing the number of shares to be burned.

```diff
function previewWithdraw(uint256 assets)
	public
	view
	override
	returns (uint256)
{
	// Calculate shares based on the specified assets' proportion of the pool
-	uint256 shares = convertToShares(assets);
+	uint256 shares = supply == 0 ? assets : assets.mulDivUp(supply, totalAssets());
	
	// Save 1 SLOAD
	uint256 _totalSupply = totalSupply;

	// Factor in additional shares to fulfill withdrawal if user is not the last to withdraw
	return
		(_totalSupply == 0 || _totalSupply - shares == 0)
			? shares
			: (shares * FEE_DENOMINATOR) /
				(FEE_DENOMINATOR - withdrawalPenalty);
}
```