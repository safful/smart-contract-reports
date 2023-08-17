## Tags

- bug
- 3 (High Risk)
- sponsor disputed
- old-submission-method
- selected for report

# [A design flaw in the case of using 2 oracles (aka PegOracle)](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/283) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/oracles/PegOracle.sol#L67-L71


# Vulnerability details

## Impact
A design flaw in the case of using 2 oracles (aka PegOracle)

## Proof of Concept
Chainlink provides price feeds denominated either in ETH or USD. But some assets don't have canonical value accessed on-chain. An example would be BTC and it's many on-chain forms like renBTC, hitBTC, WBTC, aBTC etc... For example in the case of a market on renBTC depegging from BTC value, probably a pair like renBTC/WBTC would be used (leveraging PegOracle). But even if renBTC perfectly maintains it's value to BTC, the depeg event can be triggered when WBTC significantly depreciates or appreciates against BTC value. This depeg event will be theoretically unsound since renBTC behaved as expected. The flaw comes from PegOracle because it treats both assets symmetrically.

This is also true for ETH pairs like stETH/aETH etc.. or stablecoin pairs like FRAX/MIM etc..
Of course, it should never be used like this because Chainlink provides price feeds with respect to true ETH and USD values but we have found that test files include PegOracle for stablecoin pairs...

## Tools Used
Manual

## Recommended Mitigation Steps
Support markets only for assets that have access to an oracle with price against canonical value x/ETH or x/USD. 

