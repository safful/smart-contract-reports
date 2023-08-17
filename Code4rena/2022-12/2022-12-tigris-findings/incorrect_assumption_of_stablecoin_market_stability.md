## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor acknowledged
- upgraded by judge
- H-08

# [Incorrect Assumption of Stablecoin Market Stability](https://github.com/code-423n4/2022-12-tigris-findings/issues/462) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/StableVault.sol#L39-L51
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/StableVault.sol#L60-L72


# Vulnerability details

## Impact

The `StableVault` contract attempts to group all types of stablecoins under a single token which can be minted for any of the stablecoins supported by the system as well as burned for any of them.

This is at minimum a medium-severity vulnerability as the balance sheet of the `StableVault` will consist of multiple assets which do not have a one-to-one exchange ratio between them as can be observed by trading pools such as [Curve](https://curve.fi/#/ethereum/pools/3pool/deposit) as well as the [Chainlink oracle reported prices themselves](https://data.chain.link/ethereum/mainnet/stablecoins/usdc-usd).

Given that the contract exposes a 0% slippage 1-to-1 exchange between assets that in reality have varying prices, the balance sheet of the contract can be arbitraged (especially by flash-loans) to swap an undesirable asset (i.e. USDC which at the time of submission was valued at `0.99994853` USD) for a more desirable asset (i.e. USDT which at the time of submission was valued at `1.00000000` USD) acquiring an arbitrage in the price by selling the traded asset. 

## Proof of Concept

To illustrate the issue, simply view the exchange output you would get for swapping your USDC to USDT in a stablecoin pool (i.e. CurveFi) and then proceed to [invoke `deposit`](https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/StableVault.sol#L39-L51) with your USDC asset and retrieve your [incorrectly calculated `USDT` equivalent via `withdraw`](https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/StableVault.sol#L60-L72).

The arbitrage can be observed by assessing the difference in the trade outputs and can be capitalized by selling our newly acquired `USDT` for `USDC` on the stablecoin pair we assessed earlier, ultimately ending up with a greater amount of `USDC` than we started with. This type of attack can be extrapolated by utilizing a flash-loan rather than our personal funds. 

## Tools Used

Manual review of the codebase, [Chainlink oracle resources](https://data.chain.link/popular), [Curve Finance pools](https://curve.fi/#/ethereum/pools).

## Recommended Mitigation Steps

We advise the `StableVault` to utilize Chainlink oracles for evaluating the inflow of assets instead, ensuring that all inflows and outflows of stablecoins are fairly evaluated based on their "neutral" USD price rather than their subjective on-chain price or equality assumption.