## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-30

# [User can "callOutSigned" without paying for gas by reentering "anyExecute" with virtual account](https://github.com/code-423n4/2023-05-maia-findings/issues/331) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L798-L805


# Vulnerability details

## Impact
Virtual account can perform external calls during root chain execution process. If it `callOut` at Arbitrum Branch Bridge Agent, `anyExecute` in Root Bridge Agent will be reentered. `lock` will not work if user initiated the process on another branch chain. `_payExecutionGas` will not charge gas for the reentrancy call, Meanwhile, storage variable `initialGas` and `userFeeInfo` will be deleted. As a result, no gas will be charged for the original call.
 
## Proof of Concept
```
function anyExecute(bytes calldata data)
    external
    virtual
    requiresExecutor
    returns (bool success, bytes memory result)
{
    uint256 _initialGas = gasleft();
    uint24 fromChainId;
    UserFeeInfo memory _userFeeInfo;

    if (localAnyCallExecutorAddress == msg.sender) {
        initialGas = _initialGas;
        (, uint256 _fromChainId) = _getContext();
        fromChainId = _fromChainId.toUint24();

        _userFeeInfo.depositedGas = _gasSwapIn(
            uint256(uint128(bytes16(data[data.length - PARAMS_GAS_IN:data.length - PARAMS_GAS_OUT]))), fromChainId).toUint128();
        _userFeeInfo.gasToBridgeOut = uint128(bytes16(data[data.length - PARAMS_GAS_OUT:data.length]));
    } else {
        fromChainId = localChainId;
        _userFeeInfo.depositedGas = uint128(bytes16(data[data.length - 32:data.length - 16]));
        _userFeeInfo.gasToBridgeOut = _userFeeInfo.depositedGas;
    }

    if (_userFeeInfo.depositedGas < _userFeeInfo.gasToBridgeOut) {
        _forceRevert();
        return (true, "Not enough gas to bridge out");
    }

    userFeeInfo = _userFeeInfo;

    // execution part
    ............

    if (initialGas > 0) {
        _payExecutionGas(userFeeInfo.depositedGas, userFeeInfo.gasToBridgeOut, _initialGas, fromChainId);
    }
}
```
```
function _payExecutionGas(uint128 _depositedGas, uint128 _gasToBridgeOut, uint256 _initialGas, uint24 _fromChain) internal {
    delete(initialGas);
    delete(userFeeInfo);

    if (_fromChain == localChainId) return;

    uint256 availableGas = _depositedGas - _gasToBridgeOut;
    uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasleft());
    if (minExecCost > availableGas) {
        _forceRevert();
        return;
    }

    _replenishGas(minExecCost);

    accumulatedFees += availableGas - minExecCost;
}
```
During the reentrancy call, `initialGas` will not be modified before the execution part; `_payExecutionGas` will be invoked, but it will directly return after deleting `initialGas` and `userFeeInfo`. As a result, after the execution part of original call, `_payExecutionGas` will be passed as `initialGas` is now zero.

## Tools Used
Manual

## Recommended Mitigation Steps
Store `initialGas` and `userFeeInfo` in memory as local variable inside `anyExecute`.











## Assessed type

Reentrancy