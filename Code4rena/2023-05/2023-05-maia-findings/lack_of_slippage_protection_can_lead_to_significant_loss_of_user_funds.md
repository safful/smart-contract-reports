## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-18

# [Lack of slippage protection can lead to significant loss of user funds](https://github.com/code-423n4/2023-05-maia-findings/issues/577) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L201-L210
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/libraries/PoolActions.sol#L73-L87
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L135-L149
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L353-L361


# Vulnerability details

Talos strategy contracts interact with Uniswap V3 in multiple areas of the code. However none of these interactions contain any [slippage control](https://dacian.me/defi-slippage-attacks#heading-no-slippage-parameter), meaning that the contract, and by extension all users who hold shares, can lose significant value due to illiquid pools or MEV sandwich attacks every time any of the relevant functions are called.

## Impact
[`TalosBaseStrategy#deposit`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L201-L210) is the entry point for any Talos vault, and transfers tokens from the caller to the vault to be deposited into Uniswap V3. Since it lacks slippage control, every user who interacts with any Talos vault will risk having their funds stolen by MEV bots. [`PoolActions#rerange`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/libraries/PoolActions.sol#L73-L87) is also vulnerable (which is called whenever the strategy manager wishes to rebalance pool allocation of the vault) which may lead to vault funds being at risk to the detriment of shareholders. The vault initialize function [`TalosBaseStrategy#init`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L135-L149) is vulnerable as well however only the vault owners funds would be at risk here.

## Proof of Concept
In each of the below instances, a call to Uniswap V3 is made and `amount0Min` and `amount1Min` are each set to 0, which allows for **100% slippage tolerance**. This means that the action could lead to the caller losing up to 100% of their tokens due to slippage.

[`TalosBaseStrategy#deposit`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L201-L210):
```solidity
        (liquidityDifference, amount0, amount1) = nonfungiblePositionManager.increaseLiquidity(
            INonfungiblePositionManager.IncreaseLiquidityParams({
                tokenId: _tokenId,
                amount0Desired: amount0Desired,
                amount1Desired: amount1Desired,
                amount0Min: 0, // @audit should be non-zero
                amount1Min: 0, // @audit should be non-zero
                deadline: block.timestamp
            })
        );
```

[`PoolActions#rerange`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/libraries/PoolActions.sol#L73-L87):
```solidity
        (tokenId, liquidity, amount0, amount1) = nonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(actionParams.token0),
                token1: address(actionParams.token1),
                fee: poolFee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: balance0,
                amount1Desired: balance1,
                amount0Min: 0, // @audit should be non-zero
                amount1Min: 0, // @audit should be non-zero
                recipient: address(this),
                deadline: block.timestamp
            })
```

[`TalosBaseStrategy#init`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L135-L149):
```solidity
        (_tokenId, _liquidity, amount0, amount1) = _nonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(_token0),
                token1: address(_token1),
                fee: poolFee,
                tickLower: tickLower,
                tickUpper: tickUpper,
                amount0Desired: amount0Desired,
                amount1Desired: amount1Desired,
                amount0Min: 0, // @audit should be non-zero
                amount1Min: 0, // @audit should be non-zero
                recipient: address(this),
                deadline: block.timestamp
            })
        );
```

[`TalosBaseStrategy#_withdrawAll`](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L353-L361):
```solidity
        _nonfungiblePositionManager.decreaseLiquidity(
            INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: _tokenId,
                liquidity: _liquidity,
                amount0Min: 0, // @audit should be non-zero
                amount1Min: 0, // @audit should be non-zero
                deadline: block.timestamp
            })
        );
```

## Tools Used
Manual review

## Recommended Mitigation Steps
For each vulnerable function, allow the caller to specify values for `amount0Min` and `amount1Min` instead of setting them to 0.


## Assessed type

Uniswap