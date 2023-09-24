## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-10

# [TalosBaseStrategy#init() lacks slippage protection](https://github.com/code-423n4/2023-05-maia-findings/issues/658) 

# Lines of code

https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L99-L147
https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L419-L425


# Vulnerability details

`checkDeviation` modifier purpose is to add slippage protection for increase/decrease liquidity operations. It's applied to `deposit/redeem`, `rerange/rebalance`  but `init()` is missing it. 

## Impact
There is no slippage protection on `init()`.

## Proof of Concept

In the `init()` function of TalosBaseStrategy, the following actions are performed: an initial deposit is made, a tokenId and shares are minted. 

The `_nonfungiblePositionManager.mint()` function is called with hardcoded values of `amount0Min` and `amount1Min`, both set to 0. Additionally, it should be noted that the `init()` function does not utilize the `checkDeviation` modifier, which was specifically designed to safeguard users against slippage.


```solidity
    function init(uint256 amount0Desired, uint256 amount1Desired, address receiver)
        external
        virtual
        nonReentrant
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
    ...
        (_tokenId, _liquidity, amount0, amount1) = _nonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(_token0),
                token1: address(_token1),
                fee: poolFee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: amount0Desired,
                amount1Desired: amount1Desired,
                amount0Min: 0,
                amount1Min: 0,
                recipient: address(this),
                deadline: block.timestamp
            })
        );
        ...
```

https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L99-L147

```solidity
    /// @notice Function modifier that checks if price has not moved a lot recently.
    /// This mitigates price manipulation during rebalance and also prevents placing orders when it's too volatile.
    modifier checkDeviation() {
        ITalosOptimizer _optimizer = optimizer;
        pool.checkDeviation(_optimizer.maxTwapDeviation(), _optimizer.twapDuration());
        _;
    }
```

https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L419-L425


## Tools Used

VS Code, uniswapv3book

## Recommended Mitigation Steps

Apply `checkDeviation` to `init()` function.


## Assessed type

Other