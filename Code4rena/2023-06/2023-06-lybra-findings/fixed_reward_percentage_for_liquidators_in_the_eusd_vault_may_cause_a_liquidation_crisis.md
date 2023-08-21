## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-20

# [Fixed reward percentage for liquidators in the eUSD vault may cause a liquidation crisis](https://github.com/code-423n4/2023-06-lybra-findings/issues/72) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L154-L176


# Vulnerability details

## Impact

To not lose generality, the same issue is present in the `LybraPEUSDVaultBase` contract.

Liquidations are essential for a lending protocol to maintain the over-collateralization of the protocol. Hence, when a liquidation happens, it should increment the collateral ratio of the liquidated position (make it healthier).

The `LybraEUSDVaultBase` contract has a function named `liquidation`, which is used to liquidate users whose collateral ratio is below the bad collateral ratio, which for the eUSD Vault is 150%. This function incentives liquidators with a fixed reward of 10% of the collateral being liquidated. However, the issue with the fixed compensation is that it will cause a position to get unhealthier during a liquidation when the collateral ratio is 110% or smaller. 

## Proof of Concept

Take the following example:

- USD / ETH price = 1500
- Collateral amount = 2 ether
- Debt = 2779 eUSD

The data above will give us a collateral ratio for the position of: 107.9%. The liquidator liquidates the max amount possible, which is 50% of the collateral, one ether, and takes 10% extra for its services; the final collateral ratio will be:

`((2 - 1.1) * 1500) / (2779 - 1500) = 1.055`

The position got unhealthier after the liquidation, from a collateral ratio of 107.9% to 105%. The process can be repeated until it is no longer profitable for the liquidator leading the protocol to accumulate bad debt.

## Justification

I landed medium on this finding for the following reasons:

- It has the requirement that the position must have a collateral ratio lower than 110%, which means that it was not liquidated before it reached that point. 

- Even though the above point is required for this to become an issue, the position in the example was still overcollateralized (~108%). It should not be possible to liquidate an over-collateralized position and have the consequence of making it unhealthier.

## Tools Used

Manual Review

## Recommended Mitigation Steps

When a position has a collateral ratio below 110%, the reward percentage should be adjusted accordingly instead of a fixed reward of 10%.

















## Assessed type

Math