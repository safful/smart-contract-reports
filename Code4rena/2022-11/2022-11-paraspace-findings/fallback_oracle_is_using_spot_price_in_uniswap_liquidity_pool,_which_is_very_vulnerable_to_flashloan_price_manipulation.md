## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-04

# [Fallback oracle is using spot price in Uniswap liquidity pool, which is very vulnerable to flashloan price manipulation](https://github.com/code-423n4/2022-11-paraspace-findings/issues/242) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceOracle.sol#L131
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceFallbackOracle.sol#L56
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/ParaSpaceFallbackOracle.sol#L78


# Vulnerability details

## Impact

Fallback oracle is using spot price in Uniswap liquidity pool, which is very vulnerable to flashloan price manipulation. Hacker can use flashloan to distort the price and overborrow or perform malicious liqudiation.

## Proof of Concept

In the current implementation of the paraspace oracle, if the paraspace oracle has issue, the fallback oracle is used for ERC20 token.

```solidity
/// @inheritdoc IPriceOracleGetter
function getAssetPrice(address asset)
	public
	view
	override
	returns (uint256)
{
	if (asset == BASE_CURRENCY) {
		return BASE_CURRENCY_UNIT;
	}

	uint256 price = 0;
	IEACAggregatorProxy source = IEACAggregatorProxy(assetsSources[asset]);
	if (address(source) != address(0)) {
		price = uint256(source.latestAnswer());
	}
	if (price == 0 && address(_fallbackOracle) != address(0)) {
		price = _fallbackOracle.getAssetPrice(asset);
	}

	require(price != 0, Errors.ORACLE_PRICE_NOT_READY);
	return price;
}
```


which calls:

```solidity
price = _fallbackOracle.getAssetPrice(asset);
```

whch use the spot price from Uniswap V2.

```solidity
	address pairAddress = IUniswapV2Factory(UNISWAP_FACTORY).getPair(
		WETH,
		asset
	);
	require(pairAddress != address(0x00), "pair not found");
	IUniswapV2Pair pair = IUniswapV2Pair(pairAddress);
	(uint256 left, uint256 right, ) = pair.getReserves();
	(uint256 tokenReserves, uint256 ethReserves) = (asset < WETH)
		? (left, right)
		: (right, left);
	uint8 decimals = ERC20(asset).decimals();
	//returns price in 18 decimals
	return
		IUniswapV2Router01(UNISWAP_ROUTER).getAmountOut(
			10**decimals,
			tokenReserves,
			ethReserves
		);
```

and

```solidity
function getEthUsdPrice() public view returns (uint256) {
	address pairAddress = IUniswapV2Factory(UNISWAP_FACTORY).getPair(
		USDC,
		WETH
	);
	require(pairAddress != address(0x00), "pair not found");
	IUniswapV2Pair pair = IUniswapV2Pair(pairAddress);
	(uint256 left, uint256 right, ) = pair.getReserves();
	(uint256 usdcReserves, uint256 ethReserves) = (USDC < WETH)
		? (left, right)
		: (right, left);
	uint8 ethDecimals = ERC20(WETH).decimals();
	//uint8 usdcDecimals = ERC20(USDC).decimals();
	//returns price in 6 decimals
	return
		IUniswapV2Router01(UNISWAP_ROUTER).getAmountOut(
			10**ethDecimals,
			ethReserves,
			usdcReserves
		);
}
```

Using flashloan to distort and manipulate the price is very damaging technique.

Consider the POC below.

1. the User uses 10000 amount of tokenA as collateral, each token A worth 1 USD according to the paraspace oracle. the user borrow 3 ETH, the price of ETH is 1200 USD.

2. the paraspace oracle went down, the fallback price oracle is used, the user use borrows flashloan to distort the price of the tokenA in Uniswap pool from 1 USD to 10000 USD.

3. the user's collateral position worth 1000 token X 10000 USD, and borrow 1000 ETH.

4. User repay the flashloan using the overborrowed amount and recover the price of the tokenA in Uniswap liqudity pool to 1 USD, leaving bad debt and insolvent position in Paraspace.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the project does not use the spot price in Uniswap V2, if the paraspace is down, it is safe to just revert the transaction.