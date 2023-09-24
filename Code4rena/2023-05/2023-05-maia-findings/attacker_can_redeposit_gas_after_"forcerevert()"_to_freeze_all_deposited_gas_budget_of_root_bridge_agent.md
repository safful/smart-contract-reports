## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-23

# [Attacker can redeposit gas after "forceRevert()" to freeze all deposited gas budget of Root Bridge Agent](https://github.com/code-423n4/2023-05-maia-findings/issues/337) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L1233-L1241


# Vulnerability details

## Impact
`forceRevert()` withdraws all deposited gas budget of Root Bridge Agent to ensure that failed AnyCall execution will not be charged. However, if `forceRevert()` took place during a call made by virtual account, gas can be replenished later manually. As a result, the AnyCall execution will succeed but all withdrawn gas will be frozen.

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

To implement the attack, attacker `callOutSigned` on a branch chain to bypass `lock`. On the root chain, virtual account makes three external calls.
1. `retryDeposit` at Arbitrum Branch Bridge Agent with an already executed nonce. The call will `forceRevert()` and `initialGas` will be nonzero since it has not been modified by reentering. As a result, all execution gas budget will be withdrawn.
```
function _forceRevert() internal {
    if (initialGas == 0) revert GasErrorOrRepeatedTx();

    IAnycallConfig anycallConfig = IAnycallConfig(IAnycallProxy(localAnyCallAddress).config());
    uint256 executionBudget = anycallConfig.executionBudget(address(this));

    // Withdraw all execution gas budget from anycall for tx to revert with "no enough budget"
    if (executionBudget > 0) try anycallConfig.withdraw(executionBudget) {} catch {}
}
```
2. `callOut` at Arbitrum Branch Bridge Agent. The call should succeed and `initialGas` is deleted.
```
function _payExecutionGas(uint128 _depositedGas, uint128 _gasToBridgeOut, uint256 _initialGas, uint24 _fromChain) internal {
    delete(initialGas);
    delete(userFeeInfo);

    if (_fromChain == localChainId) return;
```
3. Directly deposit a small amount of gas at Anycall Config to ensure the success of the transaction.
```
function deposit(address _account) external payable {
    executionBudget[_account] += msg.value;
    emit Deposit(_account, msg.value);
}
```
Then, the original call proceeds and `_payExecutionGas` will be skipped. The call will succeed, with all withdrawn gas budget permanently frozen. (In current implementation, ETH can be sweeped to dao address, but this is another mistake, `sweep` should transfer WETH instead.)

## Tools Used
Manual

## Recommended Mitigation Steps
Add a `msg.sender` check in `_forceRevert` to ensure local call will be directly reverted.





## Assessed type

Reentrancy