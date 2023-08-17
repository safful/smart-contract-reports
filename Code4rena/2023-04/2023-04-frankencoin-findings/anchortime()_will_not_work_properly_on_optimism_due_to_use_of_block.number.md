## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [anchorTime() will not work properly on Optimism due to use of block.number](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/914) 

# Lines of code

 https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L173


# Vulnerability details

When deploying to Optimism, Equity.anchorTime() will not be accurate due to the use of block.number. 

```
    function anchorTime() internal view returns (uint64){
        return uint64(block.number << BLOCK_TIME_RESOLUTION_BITS);
    }
```

## Impact
The inaccuracy of block.number will affect the computation of the holding duration for the votes. That will affect redeem() as the issue will cause it to deviate from the intended design of 90 days minimum holding duration (stated in comments).

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L54-L59

## Detailed Explanation
Noted that the devs have mentioned that it is conceivable that Frankencoin will be deployed on other evm chains. So it is worth reviewing the use of block.number, such that it is compatible with other chains like Optimism.

On Optimism, the `block.number` is not a reliable source of timing information and the time between each block is also different from Ethereum. This is because each transaction on L2 is placed in a separate block and blocks are not produce at a constant rate. This will cause the holding duration computation using `anchorTime()` to fluctuate. (see Optimism docs https://community.optimism.io/docs/developers/build/differences/#block-numbers-and-timestamps)



## Recommended Mitigation Steps
Consider using block.timestamp instead of block.number for more accurate measurement of time.