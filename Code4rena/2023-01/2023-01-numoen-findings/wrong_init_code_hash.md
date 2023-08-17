## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-04

# [Wrong init code hash](https://github.com/code-423n4/2023-01-numoen-findings/issues/206) 

# Lines of code

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/UniswapV2/libraries/UniswapV2Library.sol#L27


# Vulnerability details

## Impact
An init code hash is used to calculate the address of UniswapV2 pair contract. But the init code hash is not same as the latest UniswapV2 repository.

## Proof of Concept
UniswapV2Library.pairFor uses the following value as the init code hash of UniswapV2Pair. 
```
    hex"e18a34eb0e04b04f7a0ac29a6e80748dca96319b42c54d679cb821dca90c6303" // init code hash
```

But it is different from the [init code hash](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol#L24) of the uniswap v2 repository.

I tested this using one of the top UniswapV2 pairs. DAI-USDC is in the third place [here](https://v2.info.uniswap.org/pairs).

The token addresses are as follows:

DAI: 0x6B175474E89094C44Da98b954EedeAC495271d0F

USDC: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48

And the current UniswapV2Factory address is 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f [here](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/factory).

The pair address calculated is 0x6983E2Da04353C31c7C42B0EA900a40B1D5bf845. And we can't find pair contract in the address.

So I think the old version of UniswapV2Factory and pair are used here. And it can cause a risk when liquidity is not enough for the pair.


## Tools Used
Manual Review

## Recommended Mitigation Steps
Integrate the latest version of UniswapV2.