## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-07

# [redeem() beforeRedeem using the wrong owner parameter](https://github.com/code-423n4/2023-05-maia-findings/issues/730) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/base/TalosBaseStrategy.sol#L257


# Vulnerability details

## Impact
Wrong owner parameter, causing users to lose rewards

## Proof of Concept
In `TalosStrategyStaked.sol`
If the user's `shares` have changed, we need to do `flywheel.accrue()` first, which will accrue `rewards` and update the corresponding `userIndex`.
This way we can ensure the accuracy of `rewards`.
So we will call `flywheel.accrue()` beforeDeposit/beforeRedeem/transfer etc.

Take `redeem()` as an example, the code is as follows:

```solidity
contract TalosStrategyStaked is TalosStrategySimple, ITalosStrategyStaked {
...

    function beforeRedeem(uint256 _tokenId, address _owner) internal override {
        _earnFees(_tokenId);
@>      flywheel.accrue(_owner);
    }
```


But when `beforeRedeem()` is called with the wrong owner passed in, the `redeem()` code is as follows:


```solidity
    function redeem(uint256 shares, uint256 amount0Min, uint256 amount1Min, address receiver, address _owner)
        public
        virtual
        override
        nonReentrant
        checkDeviation
        returns (uint256 amount0, uint256 amount1)
    {
...
        if (msg.sender != _owner) {
            uint256 allowed = allowance[_owner][msg.sender]; // Saves gas for limited approvals.

            if (allowed != type(uint256).max) allowance[_owner][msg.sender] = allowed - shares;
        }

        if (shares == 0) revert RedeemingZeroShares();
        if (receiver == address(0)) revert ReceiverIsZeroAddress();

        uint256 _tokenId = tokenId;
@>      beforeRedeem(_tokenId, receiver);

        INonfungiblePositionManager _nonfungiblePositionManager = nonfungiblePositionManager; // Saves an extra SLOAD
        {
            uint128 liquidityToDecrease = uint128((liquidity * shares) / totalSupply);

            (amount0, amount1) = _nonfungiblePositionManager.decreaseLiquidity(
                INonfungiblePositionManager.DecreaseLiquidityParams({
                    tokenId: _tokenId,
                    liquidity: liquidityToDecrease,
                    amount0Min: amount0Min,
                    amount1Min: amount1Min,
                    deadline: block.timestamp
                })
            );

            if (amount0 == 0 && amount1 == 0) revert AmountsAreZero();

@>          _burn(_owner, shares);

            liquidity -= liquidityToDecrease;
        }    
```


From the above code, we see that the parameter is `receiver`, but the person whose shares are burn is `_owner`.

We need to accrue `_owner`, not `receiver`

This leads to a direct reduction of the user's shares without `accrue`, and the user loses the corresponding rewards


## Tools Used

## Recommended Mitigation Steps

```solidity
    function redeem(uint256 shares, uint256 amount0Min, uint256 amount1Min, address receiver, address _owner)
        public
        virtual
        override
        nonReentrant
        checkDeviation
        returns (uint256 amount0, uint256 amount1)
    {
        if (msg.sender != _owner) {
            uint256 allowed = allowance[_owner][msg.sender]; // Saves gas for limited approvals.

            if (allowed != type(uint256).max) allowance[_owner][msg.sender] = allowed - shares;
        }

        if (shares == 0) revert RedeemingZeroShares();
        if (receiver == address(0)) revert ReceiverIsZeroAddress();

        uint256 _tokenId = tokenId;
-       beforeRedeem(_tokenId, receiver);
+       beforeRedeem(_tokenId, _owner);
```


## Assessed type

Context