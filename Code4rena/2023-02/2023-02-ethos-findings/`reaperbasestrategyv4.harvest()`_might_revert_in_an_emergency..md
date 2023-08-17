## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [`ReaperBaseStrategyv4.harvest()` might revert in an emergency.](https://github.com/code-423n4/2023-02-ethos-findings/issues/730) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L109
https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperStrategyGranarySupplyOnly.sol#L200


# Vulnerability details

## Impact
`ReaperBaseStrategyv4.harvest()` might revert in an emergency if there is no position on the lending pool.

As a result, the funds might be locked inside the strategy.

## Proof of Concept
The main problem is that [Aave lending pool doesn't allow 0 withdrawals](https://github.com/aave/protocol-v2/blob/554a2ed7ca4b3565e2ceaea0c454e5a70b3a2b41/contracts/protocol/libraries/logic/ValidationLogic.sol#L60-L70).

```solidity
  function validateWithdraw(
    address reserveAddress,
    uint256 amount,
    uint256 userBalance,
    mapping(address => DataTypes.ReserveData) storage reservesData,
    DataTypes.UserConfigurationMap storage userConfig,
    mapping(uint256 => address) storage reserves,
    uint256 reservesCount,
    address oracle
  ) external view {
    require(amount != 0, Errors.VL_INVALID_AMOUNT);
```

So the below scenario would be possible.

1. After depositing and withdrawing from the Aave lending pool, the current position is 0 and the strategy is in debt.
2. It's possible that the strategy has some want balance in the contract but no position on the lending pool.
It's because `_adjustPosition()` remains the debt during reinvesting and also, there is an `authorizedWithdrawUnderlying()` for `STRATEGIST` to withdraw from the lending pool.
3. If the strategy is in an emergency, `harvest()` tries to liquidate all positions(=0 actually) and it will revert because of 0 withdrawal from Aave.
4. Also, `withdraw()` will revert at [L98](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L98) as the strategy is in the debt.

As a result, the funds might be locked inside the strategy unless the `emergency` mode is canceled.

## Tools Used
Manual Review

## Recommended Mitigation Steps
We should check 0 withdrawal in `_withdrawUnderlying()`.

```solidity
    function _withdrawUnderlying(uint256 _withdrawAmount) internal {
        uint256 withdrawable = balanceOfPool();
        _withdrawAmount = MathUpgradeable.min(_withdrawAmount, withdrawable);

        if(_withdrawAmount != 0) {
            ILendingPool(ADDRESSES_PROVIDER.getLendingPool()).withdraw(address(want), _withdrawAmount, address(this));
        }
    }
```