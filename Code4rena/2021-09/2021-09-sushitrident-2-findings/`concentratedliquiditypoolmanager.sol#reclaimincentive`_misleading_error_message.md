## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`ConcentratedLiquidityPoolManager.sol#reclaimIncentive` Misleading error message](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/54) 

# Handle

WatchPug


# Vulnerability details

https://github.com/sushiswap/trident/blob/c405f3402a1ed336244053f8186742d2da5975e9/contracts/pool/concentrated/ConcentratedLiquidityPoolManager.sol#L58

```solidity
function reclaimIncentive(
    IConcentratedLiquidityPool pool,
    uint256 incentiveId,
    uint256 amount,
    address receiver,
    bool unwrapBento
) public {
    Incentive storage incentive = incentives[pool][incentiveId];
    require(incentive.owner == msg.sender, "NOT_OWNER");
    require(incentive.expiry < block.timestamp, "EXPIRED");
    require(incentive.rewardsUnclaimed >= amount, "ALREADY_CLAIMED");
    _transfer(incentive.token, address(this), receiver, amount, unwrapBento);
    emit ReclaimIncentive(pool, incentiveId);
}
```

When the current time is before the `incentive.expiry` time, the error message should be `NOT_EXPIRED` instead of `EXPIRED`.

