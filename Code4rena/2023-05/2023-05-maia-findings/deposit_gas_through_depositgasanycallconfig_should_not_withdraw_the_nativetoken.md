## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-11

# [deposit gas through depositGasAnycallConfig should not withdraw the nativeToken](https://github.com/code-423n4/2023-05-maia-findings/issues/679) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L1219-L1222
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L848-L852


# Vulnerability details

## Impact

DepositGasAnycallConfig can deposit the gas fee externally, but here should not withdraw the nativeToken. This prevents gas from being deposited.

## Proof of Concept

There are two ways to store gas in RootBridgeAgent:

```solidity
// deposit GAS
function _manageGasOut(uint24 _toChain) internal returns (uint128) {
    uint256 amountOut;
    address gasToken;
    uint256 _initialGas = initialGas;

    if (_toChain == localChainId) {
        //Transfer gasToBridgeOut Local Branch Bridge Agent if remote initiated call.
        if (_initialGas > 0) {
            address(wrappedNativeToken).safeTransfer(getBranchBridgeAgent[localChainId], userFeeInfo.gasToBridgeOut);
        }

        return uint128(userFeeInfo.gasToBridgeOut);
    }

    if (_initialGas > 0) {
        if (userFeeInfo.gasToBridgeOut <= MIN_FALLBACK_RESERVE * tx.gasprice) revert InsufficientGasForFees();
        (amountOut, gasToken) = _gasSwapOut(userFeeInfo.gasToBridgeOut, _toChain);
    } else {
        if (msg.value <= MIN_FALLBACK_RESERVE * tx.gasprice) revert InsufficientGasForFees();
        wrappedNativeToken.deposit{value: msg.value}();
        (amountOut, gasToken) = _gasSwapOut(msg.value, _toChain);
    }

    IPort(localPortAddress).burn(address(this), gasToken, amountOut, _toChain);
    return amountOut.toUint128();
}


// pay GAS
if (localAnyCallExecutorAddress == msg.sender) {
    //Save initial gas
    initialGas = _initialGas;
}
//Zero out gas after use if remote call
if (initialGas > 0) {
    _payExecutionGas(userFeeInfo.depositedGas, userFeeInfo.gasToBridgeOut, _initialGas, fromChainId);
}
```
When localAnyCallExecutorAddress invoke anyExecute, gas fee is stored in nativeToken first, then later withdraw from nativeToken and stored into multichain. That's right

```solidity
    function depositGasAnycallConfig() external payable {
        //Deposit Gas
        _replenishGas(msg.value);
    }

    function _replenishGas(uint256 _executionGasSpent) internal {
        //Unwrap Gas
        wrappedNativeToken.withdraw(_executionGasSpent);
        IAnycallConfig(IAnycallProxy(localAnyCallAddress).config()).deposit{value: _executionGasSpent}(address(this));
    }
```

But when deposit gas directly from the outside, there is no need to interact with wrappedNativeToken, and the withdraw prevents the deposit.

## Tools Used

Manual review

## Recommended Mitigation Steps

Also add a deposit logic to depositGasAnycallConfig, or remove the withdraw logic


## Assessed type

Context