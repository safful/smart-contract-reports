## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-05

# [Replenishing gas is missing in `_payFallbackGas` of RootBridgeAgent](https://github.com/code-423n4/2023-05-maia-findings/issues/786) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L831-L846


# Vulnerability details

## Impact
`_payFallbackGas` is used to update the user deposit with the amount of gas needed to pay for the fallback function execution. 
However, it doesn't replenish gas. In other words, it doesn't deposit the executionGasSpent into AnycallConfig execution budget.

## Proof of Concept

Here is the method body.

```solidity
	function _payFallbackGas(uint32 _settlementNonce, uint256 _initialGas) internal virtual {
		//Save gasleft
		uint256 gasLeft = gasleft();

		//Get Branch Environment Execution Cost
		uint256 minExecCost = tx.gasprice * (MIN_FALLBACK_RESERVE + _initialGas - gasLeft);

		//Check if sufficient balance
		if (minExecCost > getSettlement[_settlementNonce].gasToBridgeOut) {
			_forceRevert();
			return;
		}

		//Update user deposit reverts if not enough gas
		getSettlement[_settlementNonce].gasToBridgeOut -= minExecCost.toUint128();
	}
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L831-L846

As you can see, no gas replenishing call.

`_payFallbackGas` is called at the end in `anyFallback` after reopening  user's settlement.

```solidity
	function anyFallback(bytes calldata data)
		external
		virtual
		requiresExecutor
		returns (bool success, bytes memory result)
	{
		//Get Initial Gas Checkpoint
		uint256 _initialGas = gasleft();

		//Get fromChain
		(, uint256 _fromChainId) = _getContext();
		uint24 fromChainId = _fromChainId.toUint24();

		//Save Flag
		bytes1 flag = data[0];

		//Deposit nonce
		uint32 _settlementNonce;

		/// SETTLEMENT FLAG: 1 (single asset settlement)
		if (flag == 0x00) {
			_settlementNonce = uint32(bytes4(data[PARAMS_START_SIGNED:25]));
			_reopenSettlemment(_settlementNonce);

			/// SETTLEMENT FLAG: 1 (single asset settlement)
		} else if (flag == 0x01) {
			_settlementNonce = uint32(bytes4(data[PARAMS_START_SIGNED:25]));
			_reopenSettlemment(_settlementNonce);

			/// SETTLEMENT FLAG: 2 (multiple asset settlement)
		} else if (flag == 0x02) {
			_settlementNonce = uint32(bytes4(data[22:26]));
			_reopenSettlemment(_settlementNonce);
		}
		emit LogCalloutFail(flag, data, fromChainId);

		_payFallbackGas(_settlementNonce, _initialGas);

		return (true, "");
	}
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L1177

## Tools Used
Manual analysis

## Recommended Mitigation Steps

Withdraw Gas from port, unwrap it, then call _replenishGas to top up the execution budget


## Assessed type

Other