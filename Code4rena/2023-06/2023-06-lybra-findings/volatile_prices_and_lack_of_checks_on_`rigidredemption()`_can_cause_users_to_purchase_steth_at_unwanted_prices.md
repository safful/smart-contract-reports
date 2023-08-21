## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-18

# [Volatile prices and lack of checks on `rigidRedemption()` can cause users to purchase stETH at unwanted prices](https://github.com/code-423n4/2023-06-lybra-findings/issues/221) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168


# Vulnerability details

## Impact
Volatile prices can cause issue when users try to do [`rigidRedemption`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168)

## Proof of Concept
Volatile prices can cause slippage loss, when users use `rigidRedemption()`. This function takes PeUSD (stable coin) amount and converts it to WstETH/stETH (variable price). Unfortunately `rigidRedemption()` does not include `timestamp` or `minAmount` received , meaning this trade can be executed later in time and at a different price than user previously expected.

**Example:**
- provider has 100 **wstETH** and **wstETH** price is 2000$
- user wants to buy 10 **wstETH** and has 20 000 in PeUSD, so he calls [`rigidRedemption`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168)
- Now due to congestion on **ETH**, and volatile prices the transaction could remain stuck in the mempool for a long time
- Finally the transaction gets executed, but now **wstETH **price is 2100$, not the original 2000$, so the user receives 9,52 **wstETH** instead of 10 (not counting fees)!
## Tools Used
Manual Review

## Recommended Mitigation Steps
Because of this scenario and other like it, it is recommended to use some sort of slippage protection when users execute trades.
```jsx
    function rigidRedemption(address provider, uint256 eusdAmount,uint256 minAmountReceived) external virtual {
        depositedAsset[provider] -= collateralAmount;
        totalDepositedAsset -= collateralAmount;
+       require(minAmountReceived <= collateralAmount);
        collateralAsset.transfer(msg.sender, collateralAmount);
        emit RigidRedemption(msg.sender, provider, eusdAmount, collateralAmount, block.timestamp);
    }
```


## Assessed type

MEV