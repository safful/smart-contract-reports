## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-28

# [Lack of support for ERC20 token that is not 18 decimals](https://github.com/code-423n4/2023-01-astaria-findings/issues/129) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L66
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L73


# Vulnerability details

## Impact

Lack of support for ERC20 token that is not 18 decimals in PublicVault.sol

## Proof of Concept

We need to look into the PublicVault.sol implementation

```solidity
contract PublicVault is VaultImplementation, IPublicVault, ERC4626Cloned {
```

the issue that is the decimal precision in the PublicVault is hardcoded to 18

```solidity
function decimals()
	public
	pure
	virtual
	override(IERC20Metadata)
returns (uint8)
{
	return 18;
}
```

According to 

https://eips.ethereum.org/EIPS/eip-4626

> Although the convertTo functions should eliminate the need for any use of an EIP-4626 Vault’s decimals variable, it is still strongly recommended to mirror the underlying token’s decimals if at all possible, to eliminate possible sources of confusion and simplify integration across front-ends and for other off-chain users.

The solmate ERC4626 implementation did mirror the underlying token decimals

https://github.com/transmissions11/solmate/blob/3998897acb502fa7b480f505138a6ae1842e8d10/src/mixins/ERC4626.sol#L38

```solidity
constructor(
	ERC20 _asset,
	string memory _name,
	string memory _symbol
) ERC20(_name, _symbol, _asset.decimals()) {
	asset = _asset;
}
```

but the token decimals is over-written to 18 decimals.

https://github.com/d-xo/weird-erc20#low-decimals

Some tokens have low decimals (e.g. USDC has 6). Even more extreme, some tokens like Gemini USD only have 2 decimals.

For example, if the underlying token is USDC and has 6 decimals, the convertToAssets() function will be broken.

https://github.com/transmissions11/solmate/blob/3998897acb502fa7b480f505138a6ae1842e8d10/src/mixins/ERC4626.sol#L130

```solidity
function convertToAssets(uint256 shares) public view virtual returns (uint256) {
	uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

	return supply == 0 ? shares : shares.mulDivDown(totalAssets(), supply);
}
```

the totalSupply is in 18 deimals, but the totalAssets is in 6 deciimals, but the totalSupply should be 6 decimals as well to match the underlying token precision.

There are place tha the code assume the token is 18 decimals, if the token is not 18 decimals, the logic for liquidatoin ratio calculation is broken as well because the hardcoded 1e18 is used.

```solidity
s.liquidationWithdrawRatio = proxySupply
.mulDivDown(1e18, totalSupply())
.safeCastTo88();

currentWithdrawProxy.setWithdrawRatio(s.liquidationWithdrawRatio);
uint256 expected = currentWithdrawProxy.getExpected();

unchecked {
if (totalAssets() > expected) {
  s.withdrawReserve = (totalAssets() - expected)
	.mulWadDown(s.liquidationWithdrawRatio)
	.safeCastTo88();
} else {
  s.withdrawReserve = 0;
}
}
_setYIntercept(
s,
s.yIntercept -
  totalAssets().mulDivDown(s.liquidationWithdrawRatio, 1e18)
);
```

and In the claim function for WithdrawProxy.sol

```solidity
if (balance < s.expected) {
  PublicVault(VAULT()).decreaseYIntercept(
	(s.expected - balance).mulWadDown(1e18 - s.withdrawRatio)
  );
} else {
  PublicVault(VAULT()).increaseYIntercept(
	(balance - s.expected).mulWadDown(1e18 - s.withdrawRatio)
  );
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol make the PublicVault.sol decimal match the underlying token decimals.