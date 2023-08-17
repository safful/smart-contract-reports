## Tags

- bug
- grade-a
- QA (Quality Assurance)
- selected for report
- MR-M-07

# [Lack of Initialization and scope check for slippage](https://github.com/code-423n4/2023-06-xeth-mitigation-findings/issues/16) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/add-xeth/src/AMO2.sol#L145
https://github.com/code-423n4/2023-05-xeth/blob/add-xeth/src/AMO2.sol#L430-L442


# Vulnerability details

1. 
The `upSlippage` doesn't have a default value. So if rebalance is called before the admin setSlippage, the `upSlippage` is always zero.

2. 
The `AMO2.setSlippage` function does not implement the scope check in the comments:
```
@notice The new maximum slippage must be between 0.06% and 15% (in basis points).
```
It only limits the slippages are not > 100%:
```
if (newUpSlippage >= BASE_UNIT || newDownSlippage >= BASE_UNIT)
          revert InvalidSetSlippage();
```


## Assessed type

Invalid Validation