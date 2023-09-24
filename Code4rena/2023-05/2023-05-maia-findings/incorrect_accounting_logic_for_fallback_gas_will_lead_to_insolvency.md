## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor confirmed
- M-31

# [Incorrect accounting logic for fallback gas will lead to insolvency](https://github.com/code-423n4/2023-05-maia-findings/issues/313) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L823
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L1044


# Vulnerability details

## Impact
Incorrect accounting logic for fallback gas will lead to insolvency.

## Proof of Concept
    // on root chain
    function _payExecutionGas(uint128 _depositedGas, uint128 _gasToBridgeOut, uint256 _initialGas, uint24 _fromChain) internal {
        ......
        uint256 availableGas = _depositedGas - _gasToBridgeOut;
        uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasleft());

        if (minExecCost > availableGas) {
            _forceRevert();
            return;
        }

        _replenishGas(minExecCost);

        //Account for excess gas
        accumulatedFees += availableGas - minExecCost;
    }

    // on branch chain
    function _payFallbackGas(uint32 _depositNonce, uint256 _initialGas) internal virtual {
        ......
        IPort(localPortAddress).withdraw(address(this), address(wrappedNativeToken), minExecCost);
        wrappedNativeToken.withdraw(minExecCost);
        _replenishGas(minExecCost);
    }

As above, when paying execution gas on the root chain, the excessive gas is added to accumulatedFees. So theoretically all deposited gas is used up and no gas has been reserved for `anyFallback` on the branch chain. The withdrawl in `_payFallbackGas` on branch chain will cause insolvency.

    // on branch chain
    function _payExecutionGas(address _recipient, uint256 _initialGas) internal virtual {
        ......
        uint256 gasLeft = gasleft();
        uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

        if (minExecCost > gasRemaining) {
            _forceRevert();
            return;
        }

        _replenishGas(minExecCost);

        //Transfer gas remaining to recipient
        SafeTransferLib.safeTransferETH(_recipient, gasRemaining - minExecCost);
        ......
        }
    }

    // on root chain
    function _payFallbackGas(uint32 _settlementNonce, uint256 _initialGas) internal virtual {
        uint256 gasLeft = gasleft();
        uint256 minExecCost = tx.gasprice * (MIN_FALLBACK_RESERVE + _initialGas - gasLeft);

        if (minExecCost > getSettlement[_settlementNonce].gasToBridgeOut) {
            _forceRevert();
            return;
        }

        getSettlement[_settlementNonce].gasToBridgeOut -= minExecCost.toUint128();
    }

As above, when paying execution gas on the branch chain, the excessive gas has be sent to the recipent. So therotically all deposited gas is used up and no gas has been reserved for `anyFallback` on the root chain. `_payFallbackGas` does not `_replenishGas`, which will cause insolvency of gas budget in `AnycallConfig`.
## Tools Used
Manual

## Recommended Mitigation Steps
Deduct fallback gas from deposited gas.


## Assessed type

Context