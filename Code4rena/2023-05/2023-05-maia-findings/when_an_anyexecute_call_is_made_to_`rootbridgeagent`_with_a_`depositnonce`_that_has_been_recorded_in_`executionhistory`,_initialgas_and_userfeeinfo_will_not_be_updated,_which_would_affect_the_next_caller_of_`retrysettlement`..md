## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-12

# [When an anyExecute call is made to `RootBridgeAgent` with a `depositNonce` that has been recorded in `executionHistory`, initialGas and userFeeInfo will not be updated, which would affect the next caller of `retrySettlement`.](https://github.com/code-423n4/2023-05-maia-findings/issues/676) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L873-L890
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L922
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L246
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L571


# Vulnerability details

## Impact
Wrong userFeeInfo will be used when `retrySettlement` is called directly.

## Proof of Concept
Here is `retrySettlement` function:

```solidity
function retrySettlement(
    uint32 _settlementNonce,
    uint128 _remoteExecutionGas
) external payable {
    //Update User Gas available.
    if (initialGas == 0) {
        userFeeInfo.depositedGas = uint128(msg.value);
        userFeeInfo.gasToBridgeOut = _remoteExecutionGas;
    }
    //Clear Settlement with updated gas.
    _retrySettlement(_settlementNonce);
}
```

The assumption here is that if initialGas is not 0, then `retrySettlement` is being called by `RootBridgeAgent#anyExecute`, which has already set values for initialGas and userFeeInfo(which would later be deleted at end of the anycall function), but if it is 0, then retrySettlement is being called directly by a user, so user should specify `_remoteExecutionGas` and send some `msg.value` with the call, which would make up the userFeeInfo.
But this assumption is not completely correct because whenever `RootBridgeAgent#anyExecute` is called with a depositNonce that has been recorded in executionHistory, the function returns early, which prevents other parts of the anyExecute function from being executed.
At the beginning of anyExecute, initialGas and userFeeInfo values are set and at the end of anyExecute call, if initialGas>0, `_payExecutionGas` sets initialGas and userFeeInfo to 0.
So when the function returns earlier, before `_payExecutionGas` is called, initialGas and userFeeInfo are not updated.
If a user calls `retrySettlement` immediately after that, the call will use a wrong userFeeInfo(i.e. userFeeInfo set when anyExecute was called with a depositNonce that has already been recorded) because initialGas!=0. Whereas, it was meant to use values sent by caller of `retrySettlement`

Looking at a part of `_manageGasOut` logic which is called in `_retrySettlement`,

```solidity

if (_initialGas > 0) {
    if (
        userFeeInfo.gasToBridgeOut <= MIN_FALLBACK_RESERVE * tx.gasprice
    ) revert InsufficientGasForFees();
    (amountOut, gasToken) = _gasSwapOut(
        userFeeInfo.gasToBridgeOut,
        _toChain
    );
} else {
    if (msg.value <= MIN_FALLBACK_RESERVE * tx.gasprice)
        revert InsufficientGasForFees();
    wrappedNativeToken.deposit{value: msg.value}();
    (amountOut, gasToken) = _gasSwapOut(msg.value, _toChain);
}
```

This could cause one of these:

- User's `retrySettlement` call would revert if userFeeInfo.gasToBridgeOut(which user does not have control over) is less than `MIN_FALLBACK_RESERVE * tx.gasprice`
- User's call passes without him sending any funds, so he makes a free `retrySettlement` transaction


## Tools Used
Manual Review

## Recommended Mitigation Steps

Consider implementing one of these:

- restrict retrySettlement to only be called by AgentExecutor
- or delete initialGas and userFeeInfo before return is called if nonce has been executed before:

```solidity
//Check if tx has already been executed
if (executionHistory[fromChainId][nonce]) {
    _forceRevert();
    delete initialGas;
    delete userFeeInfo;
    //Return true to avoid triggering anyFallback in case of `_forceRevert()` failure
    return (true, "already executed tx");
}
```



## Assessed type

Error