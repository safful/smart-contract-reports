## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [UniswapV3Oracle: No events emitted for setUniPriceConverter, setTwapPeriod, setMinObservations](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/53) 

# Handle

greiart


# Vulnerability details

## Impact

No impact but personally, I think it's good practice to emit an event whenever you update the state of the contract via a setter function.

## Referenced Codelines

[https://github.com/code-423n4/2021-07-wildcredit/blob/main/contracts/UniswapV3Oracle.sol#L65-L75](https://github.com/code-423n4/2021-07-wildcredit/blob/main/contracts/UniswapV3Oracle.sol#L65-L75)

