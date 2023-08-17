## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-07

# [Incorrect slippage check in the AMO2.rebalanceUp can be attacked by MEV](https://github.com/code-423n4/2023-05-xeth-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/main/src/AMO2.sol#L323-L325
https://github.com/code-423n4/2023-05-xeth/blob/main/src/AMO2.sol#L335-L351


# Vulnerability details

## Impact
The `AMO2.rebalanceUp` uses `AMO2.bestRebalanceUpQuote` function to avoid MEV attack when removing liquidity with only one coin. But the `bestRebalanceUpQuote` does not calculate the slippage correctly in this case, which is vulnerable to be attacked by MEV sandwich.

## Proof of Concept
`AMO2.rebalanceUp` can be executed sucessfully only if the `preRebalanceCheck` returns true.
```solidity
        bool isRebalanceUp = preRebalanceCheck();
        if (!isRebalanceUp) revert RebalanceUpNotAllowed();
``` 
So as the logic in the `preRebalanceCheck`, if `isRebalanceUp = ture`, the token balances in the curve should satisfy the following condition
```
xETHBal / (stETHBal + xETHBal) > REBALANCE_UP_THRESHOLD 
```
The default value of REBALANCE_UP_THRESHOLD is 0.75 :
```
uint256 public REBALANCE_UP_THRESHOLD = 0.75E18;
```

It's always greater than 0.5 . So when removing liquidity for only xETH, which is the `rebalanceUp` function actually doing, the slippage will be positive instead of negative. You will receive more than 1 uint of xETH when you burn the lp token of 1 virtual price.  

But the slippage caculation is incorrect in the `bestRebalanceUpQuote`:
```solidity
    // min received caculate in bestRebalanceUpQuote
        bestQuote.min_xETHReceived = applySlippage(
            (vp * defenderQuote.lpBurn) / BASE_UNIT
        );

    // applySlippage function
    function applySlippage(uint256 amount) internal view returns (uint256) {
        return (amount * (BASE_UNIT - maxSlippageBPS)) / BASE_UNIT;
    }
```
It uses the target amount subtracted slippage as the min token amount received. It is equivalent to expanding the slippage in the opposite direction. It makes the `contractQuote` can't protect the `defenderQuote`, increasing the risk of a large sandwich.

## Tools Used
Manual review
## Recommended Mitigation Steps
rebalanceUp and rebalanceDown should use different slippage calculation methods in the `applySlippage`.


## Assessed type

MEV