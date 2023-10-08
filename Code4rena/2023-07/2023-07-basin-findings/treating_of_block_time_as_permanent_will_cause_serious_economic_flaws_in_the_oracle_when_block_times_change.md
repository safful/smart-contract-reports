## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [Treating of BLOCK_TIME as permanent will cause serious economic flaws in the oracle when block times change](https://github.com/code-423n4/2023-07-basin-findings/issues/176) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/pumps/MultiFlowPump.sol#L39


# Vulnerability details

## Description

Pumps receive the chain BLOCK_TIME in the constructor. In every update, it is used to calculate the `blocksPassed` variable, which determines what is the maximum change in price (done in `_capReserve()`).

The issue is that BLOCK_TIME is an immutable variable in the pump, which is immutable in the Well, meaning it is basically set in stone and can only be changed through a Well redeploy and liquidity migration (very long cycle).
However, BLOCK_TIME actually changes every now and then, especially in L2s.For example, the recent Bedrock upgrade in Optimism completely [changed](https://community.optimism.io/docs/developers/bedrock/differences/#the-evm) the block time generation. It is very clear this will happen many times over the course of Basin's lifetime.

When a wrong BLOCK_TIME is used, the `_capReserve()` function will either limit price changes too strictly, or too permissively. In the too strict case, this would cause larger and large deviations between the oracle pricing and the real market prices, leading to large arb opportunities. In the too permissive case, the function will not cap changes like it is meant too, making the oracle more manipulatable than the economic model used when deploying the pump.

## Impact

Treating of BLOCK_TIME as permanent will cause serious economic flaws in the oracle when block times change.

## Tools Used

Manual audit

## Recommended Mitigation Steps

The BLOCK_TIME should be changeable, given a long enough freeze period where LPs can withdraw their tokens if they are unsatisfied with the change.


## Assessed type

Timing