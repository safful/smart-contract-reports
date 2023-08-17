## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- sponsor disputed
- selected for report
- M-14

# [Two day low oracle used in `Market.liquidate()` makes the system highly at risk in an oracle attack ](https://github.com/code-423n4/2022-10-inverse-findings/issues/469) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L596
https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L594
https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L597


# Vulnerability details

## Impact
Usage of the 2 day low exchange rate when trying to liquidate is highly risky as it incentives even more malicious agents to control the price feed for a short period of time. By controlling shortly the feed, it puts at risk any debt opened for a 2 day period + the collateral released will be overshoot during the liquidation

## Proof of Concept
The attack can be done by either an attack directly on the feed to push bad data, or in the case of Chainlink manipulating for a short period of time the markets to force an update from Chainlink. Then when either of the attacks has been made the attacker call `Oracle.getPrice()`. It then gives a 2 day period to the attacker (and any other agent who wants to liquidate) to liquidate any escrow. 

This has a second drawback, we see that we use the same value at line 596, which is used to compute the liquidator reward (l.597), leading to more collateral released than expected. For instance manipulating once the feed and bring the ETH/USD rate to 20 instead of 2000, liquidator will earn 100 more than he should have had.

## Tools Used

## Recommended Mitigation Steps
Instead of using the 2 day lowest price during the liquidation, the team could either take the current oracle price, while still using the 2 day period for any direct agent interaction to minimise attacks both from users side and liquidators side