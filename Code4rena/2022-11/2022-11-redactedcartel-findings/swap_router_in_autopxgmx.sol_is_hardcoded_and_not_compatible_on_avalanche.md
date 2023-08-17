## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-05

# [SWAP_ROUTER in AutoPxGmx.sol is hardcoded and not compatible on Avalanche](https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/132) 

# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L18
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L96
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L268


# Vulnerability details

## Impact

I want to quote from the doc:

```solidity
- Does it use a side-chain? Yes
- If yes, is the sidechain evm-compatible? Yes, Avalanche
```

this shows that the projects is intended to support Avalanche side-chain.

SWAP_ROUTER in the AutoPxGmx.sol is hardcoded as:

```solidity
IV3SwapRouter public constant SWAP_ROUTER =
	IV3SwapRouter(0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45);
```

but this is address is the Uniswap V3 router address in arbitrium, but it is a EOA address in Avalanche,

https://snowtrace.io/address/0x68b3465833fb72a70ecdf485e0e4c7bd8665fc45

then the AutoPxGmx.sol is not working in Avalanche.

```solidity
gmxAmountOut = SWAP_ROUTER.exactInputSingle(
	IV3SwapRouter.ExactInputSingleParams({
		tokenIn: address(gmxBaseReward),
		tokenOut: address(gmx),
		fee: fee,
		recipient: address(this),
		amountIn: gmxBaseRewardAmountIn,
		amountOutMinimum: amountOutMinimum,
		sqrtPriceLimitX96: sqrtPriceLimitX96
	})
);
```


## Proof of Concept

the code below revert because the EOA address on Avalanche does not have exactInputSingle method in compound method.

```solidity
gmxAmountOut = SWAP_ROUTER.exactInputSingle(
	IV3SwapRouter.ExactInputSingleParams({
		tokenIn: address(gmxBaseReward),
		tokenOut: address(gmx),
		fee: fee,
		recipient: address(this),
		amountIn: gmxBaseRewardAmountIn,
		amountOutMinimum: amountOutMinimum,
		sqrtPriceLimitX96: sqrtPriceLimitX96
	})
);
```

```solidity
/**
	@notice Compound pxGMX rewards
	@param  fee                    uint24   Uniswap pool tier fee
	@param  amountOutMinimum       uint256  Outbound token swap amount
	@param  sqrtPriceLimitX96      uint160  Swap price impact limit (optional)
	@param  optOutIncentive        bool     Whether to opt out of the incentive
	@return gmxBaseRewardAmountIn  uint256  GMX base reward inbound swap amount
	@return gmxAmountOut           uint256  GMX outbound swap amount
	@return pxGmxMintAmount        uint256  pxGMX minted when depositing GMX
	@return totalFee               uint256  Total platform fee
	@return incentive              uint256  Compound incentive
 */
function compound(
	uint24 fee,
	uint256 amountOutMinimum,
	uint160 sqrtPriceLimitX96,
	bool optOutIncentive
)
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the project not hardcode the SWAP_ROUTER in AutoPxGmx.sol,
can pass this parameter in the constructor.
