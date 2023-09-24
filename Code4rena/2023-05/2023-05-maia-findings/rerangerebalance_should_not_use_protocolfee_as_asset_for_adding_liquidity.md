## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-35

# [Rerange/rebalance should not use protocolFee as asset for adding liquidity](https://github.com/code-423n4/2023-05-maia-findings/issues/80) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/libraries/PoolActions.sol#L55-L88
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/libraries/PoolActions.sol#L90-L103


# Vulnerability details

## Impact
Account of protocol fee is broken, because tokens of `protocolFee0` and `protocolFee1` are used while rerange/rebalance to add liquidity. At the same time this variables `protocolFee0` and `protocolFee1` are not updated, and de facto contract doesn't have protocol fee on balance

## Proof of Concept
Function rerange is used both in rerange and in rebalance:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/strategies/TalosStrategySimple.sol#L30-L46
```solidity
    function doRerange() internal override returns (uint256 amount0, uint256 amount1) {
        (tickLower, tickUpper, amount0, amount1, tokenId, liquidity) = nonfungiblePositionManager.rerange(
            PoolActions.ActionParams(pool, optimizer, token0, token1, tickSpacing), poolFee
        );
    }

    function doRebalance() internal override returns (uint256 amount0, uint256 amount1) {
        int24 baseThreshold = tickSpacing * optimizer.tickRangeMultiplier();

        PoolActions.ActionParams memory actionParams =
            PoolActions.ActionParams(pool, optimizer, token0, token1, tickSpacing);

        PoolActions.swapToEqualAmounts(actionParams, baseThreshold);

        (tickLower, tickUpper, amount0, amount1, tokenId, liquidity) =
            nonfungiblePositionManager.rerange(actionParams, poolFee);
    }
```
Let's have a look at this function. This function calls `getThisPositionTicks` to get the amounts balance0 and balance1 of tokens to addLiquidity:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/libraries/PoolActions.sol#L56-L88
```solidity
    function rerange(
        INonfungiblePositionManager nonfungiblePositionManager,
        ActionParams memory actionParams,
        uint24 poolFee
    )
        internal
        returns (int24 tickLower, int24 tickUpper, uint256 amount0, uint256 amount1, uint256 tokenId, uint128 liquidity)
    {
        int24 baseThreshold = actionParams.tickSpacing * actionParams.optimizer.tickRangeMultiplier();

        uint256 balance0;
        uint256 balance1;
        (balance0, balance1, tickLower, tickUpper) = getThisPositionTicks(
            actionParams.pool, actionParams.token0, actionParams.token1, baseThreshold, actionParams.tickSpacing
        );
        emit Snapshot(balance0, balance1);

        (tokenId, liquidity, amount0, amount1) = nonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(actionParams.token0),
                token1: address(actionParams.token1),
                amount0Desired: balance0,
                amount1Desired: balance1,
                ...
            })
        );
    }
```

Mistake is in function `getThisPositionTicks()` because it returns actual token balance of Strategy contract:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/libraries/PoolActions.sol#L90-L103
```solidity
    function getThisPositionTicks(
        IUniswapV3Pool pool,
        ERC20 token0,
        ERC20 token1,
        int24 baseThreshold,
        int24 tickSpacing
    ) private view returns (uint256 balance0, uint256 balance1, int24 tickLower, int24 tickUpper) {
        // Emit snapshot to record balances
        balance0 = token0.balanceOf(address(this));
        balance1 = token1.balanceOf(address(this));

        //Get exact ticks depending on Optimizer's balances
        (tickLower, tickUpper) = pool.getPositionTicks(balance0, balance1, baseThreshold, tickSpacing);
    }
```

Returns actual balance which consists of 2 parts: protocolFee and users' funds. Rerange must use users' funds, but not protocolFee.

Suppose following scenario:
1) User added 1000 tokens of liquidity
2) This liquidity generated 100 tokens of fee, 50 of which is protocolFee
3) Rerange is called. After removing liquidity contract has 1000 + 100 tokens balance. And contract add liquidity of whole balance - 1100 tokens
4) Function collectFee doesn't work because actual balance is less than withdrawing amount. And protocol loses profit:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/base/TalosBaseStrategy.sol#L394-L415
```solidity
    function collectProtocolFees(uint256 amount0, uint256 amount1) external nonReentrant onlyOwner {
        uint256 _protocolFees0 = protocolFees0;
        uint256 _protocolFees1 = protocolFees1;

        if (amount0 > _protocolFees0) {
            revert Token0AmountIsBiggerThanProtocolFees();
        }
        if (amount1 > _protocolFees1) {
            revert Token1AmountIsBiggerThanProtocolFees();
        }
        ERC20 _token0 = token0;
        ERC20 _token1 = token1;
        uint256 balance0 = _token0.balanceOf(address(this));
        uint256 balance1 = _token1.balanceOf(address(this));
        require(balance0 >= amount0 && balance1 >= amount1);
        if (amount0 > 0) _token0.transfer(msg.sender, amount0);
        if (amount1 > 0) _token1.transfer(msg.sender, amount1);

        protocolFees0 = _protocolFees0 - amount0;
        protocolFees1 = _protocolFees1 - amount1;
        emit RewardPaid(msg.sender, amount0, amount1);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
I suggest using different address for protocolFee. Transfer all protocolFee tokens away from this contract to not mix it with users' assets. Create a contract like "ProtocolFeeReceiver.sol" and make force transfer of tokens when Strategy gets fee

Also want to note that in forked parent project SorbettoFragola it is implemented via burnExactLiquidity
https://github.com/Popsicle-Finance/SorbettoFragola/blob/9fb31b74f19005d86a78abc758553e7914e7ba49/SorbettoFragola.sol#L458-L483








## Assessed type

Math