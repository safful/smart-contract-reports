## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [withdrawAllAndUnwrap() the clpToken transfer to AMO.sol may be locked in the contract](https://github.com/code-423n4/2023-05-xeth-findings/issues/6) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/CVXStaker.sol#L177


# Vulnerability details

## Impact
in `withdrawAllAndUnwrap()` 
the clpToken transfer to AMO.sol may be locked in the contract

## Proof of Concept
`withdrawAllAndUnwrap()` You can specify `sendToOperator==true` to transfer the `clpToken` to `operator`

The code is as follows:
```solidity
    function withdrawAllAndUnwrap(
        bool claim,
        bool sendToOperator
    ) external onlyOwner {
        IBaseRewardPool(cvxPoolInfo.rewards).withdrawAllAndUnwrap(claim);
        if (sendToOperator) {
            uint256 totalBalance = clpToken.balanceOf(address(this));
            clpToken.safeTransfer(operator, totalBalance); //<------@audit transfer to operator (AMO)
        }
    }
```

current protocols, `operator` is currently set to `AMO.sol` as normal

But `AMO.sol` doesn't have any way to use the transferred `clpToken`
The reason is that in AMO.sol, the method that transfers the `clpToken`, the number of transfers is from the newly generated `clpToken` from `curvePool`

It doesn't include `clpToken` that already exists in `AMO.sol` contract, for example (rebalanceDown/addLiquidity/addLiquidityOnlyStETH)

example `rebalanceDown`:
```solidity
    function rebalanceDown(
        RebalanceDownQuote memory quote
    )
...

        lpAmountOut = curvePool.add_liquidity(amounts, quote.minLpReceived);

        IERC20(address(curvePool)).safeTransfer(
            address(cvxStaker),
            lpAmountOut //<---------@audit this clpToken from curvePool
        );
        cvxStaker.depositAndStake(lpAmountOut);    
```


So the `clpToken` transferred to 'AMO.sol' by `withdrawAllAndUnwrap()` will stays in the AMO contract and it is be locked.



## Tools Used

## Recommended Mitigation Steps

modify `withdrawAllAndUnwrap()` , directly transfer to `msg.sender`.


## Assessed type

Context