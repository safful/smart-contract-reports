## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor disputed
- M-05

# [`upgradeProtocol` can create Peg Risk via Oracle Price Arbitrage](https://github.com/code-423n4/2023-02-ethos-findings/issues/708) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LUSDToken.sol#L160-L181


# Vulnerability details

### Impact
`upgradeProtocol` is meant to enable a new version of the protocol while retaining the same LUSD token

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LUSDToken.sol#L160-L181

```solidity
    function upgradeProtocol(
```

In case of a migration, with the same collateral, but a new oracle, the system could open up to arbitrage between the two oracles via redemptions, allowing to extract value from the difference between the 2 prices.

This is because each oracle (e.g. chainlink), can change it's price based on two aspects:
- Hearbeat -> Maximum amount of time before the feed is updated
- Threshold -> % change at which the price is changed no matter what

In case of the oracle being different, for example having a different heartbeat setting, or simply having a different cadence (e.g. one refreshes at noon the other at 3 pm), the difference can open up to Arbitrage Strategies that can potentially increase risk to the system


### Arbitrage through Redemptions Explanation

The fact that that older version of the protocol can burn means they could allow for redemption arbitrage, leaking value.

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LUSDToken.sol#L366-L367

```solidity
        // old versions of the protocol may still burn

```

Burning of tokens can be performed via two operations:
- Repayment
- Redemptions

Repayment seems to be safest options and it's hard to imagine a scenario for exploit.

If the oracle offers a different price for redemptions, that can crate an incentive to go redeem against the older system, and since the older system cannot create new Troves, the CR for it could suffer.

The way in which this get's problematic is if there's positions that risk becoming under-collateralized in the old system and the debt from those positions is used to redeem against better collateralized positions on the new migrated system

This would create an economic incentive to leave the bad debt in older system as the new one is offering a more profitable opportunity


### Additional Resources

An example of desynch is what happened to a Gearbox Ninja, that got liquidated due to hearbeat differences

https://twitter.com/gearbox_intern/status/1587957233605918721

### Remediation Steps

It will be best to ensure that a collateral is either in the old system, or on the new system, and if the same collateral is in both version, I believe the Oracle must be the same as to avoid inconsistent pricing.

It may also be best to change the migration pattern to one based on Zaps, which will offer good UX but reduce risk to the LUSD peg dynamic