## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-10

# [The user is enforced to overpay for the fallback gas when `retryDeposit`](https://github.com/code-423n4/2023-05-maia-findings/issues/710) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L319-L328


# Vulnerability details

## Impact
`BranchBridgeAgent.retryDeposit` is used to top up a previous deposit and perform a call afterwards.  A modifier `requiresFallbackGas` is added to the method to verifiy enough gas is deposited to pay for an eventual fallback call. The same is done when creating a new deposit.

- retryDeposit
```
	function retryDeposit(
		bool _isSigned,
		uint32 _depositNonce,
		bytes calldata _params,
		uint128 _remoteExecutionGas,
		uint24 _toChain
	) external payable lock requiresFallbackGas {
		//Check if deposit belongs to message sender
		if (getDeposit[_depositNonce].owner != msg.sender) revert NotDepositOwner();

	.
	.
	.
	.
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L319-L328

- An example of a new deposit/call
```solidity
// One example
    function callOutSignedAndBridge(bytes calldata _params, DepositInput memory _dParams, uint128 _remoteExecutionGas)
        external
        payable
        lock
        requiresFallbackGas
    {

// Another one
    function callOutSignedAndBridgeMultiple(
        bytes calldata _params,
        DepositMultipleInput memory _dParams,
        uint128 _remoteExecutionGas
    ) external payable lock requiresFallbackGas {

```


Let's have a look at the modifier `requiresFallbackGas`
```
    /// @notice Modifier that verifies enough gas is deposited to pay for an eventual fallback call.
    modifier requiresFallbackGas() {
        _requiresFallbackGas();
        _;
    }

    /// @notice Verifies enough gas is deposited to pay for an eventual fallback call. Reuse to reduce contract bytesize.
    function _requiresFallbackGas() internal view virtual {
        if (msg.value <= MIN_FALLBACK_RESERVE * tx.gasprice) revert InsufficientGas();
    }
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1404-L1412

It checks if the msg.value (deposited gas) is sufficient. This is used for both a new deposit and topping up an existing deposit. For a new deposit it makes sense. However, for topping up an existing deposit it doesn't consider the old deposited amount which enforce the user to overpay for the gas when `retryDeposit`. Please have a look at the PoC to get a clearer picture.


## Proof of Concept

Imagine the following scenario:
- User **Bob** makes a request by `BaseBranchRouter.callOutAndBridge` with msg.value 0.1 ETH (**deposited gas is 0.1 ETH**) assuming the **cost of MIN_FALLBACK_RESERVE is 0.1 ETH**.
- This calls `BranchBridgeAgent.performCallOutAndBridge` 
- BranchBridgeAgent creates deposit and send Cross-Chain request by calling `AnycallProxy.anyCall`
- Now AnyCall Executor calls `RootBridgeAgent.anyExecute`
- Let's say `RootBridgeAgent.anyExecute` couldn't complete due to insufficient available gas.
```solidity
	//Get Available Gas
	uint256 availableGas = _depositedGas - _gasToBridgeOut;

	//Get Root Environment Execution Cost
	uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasleft());

	//Check if sufficient balance
	if (minExecCost > availableGas) {
		_forceRevert();
		return;
	}
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L810-L817

- Notice that this _forceReverts and doesn't revert directly. This is to avoid triggering the fallback in BranchBridgeAgent (below an explanation of _forceRevert).
- Let's assume that the additional required deposit was 0.05 ETH
- So now  **Bob** should top up the deposit with 0.05 ETH.
- **Bob** calls `BranchBridgeAgent.retryDeposit` and since there is `requiresFallbackGas` modifier, he has to pass at least 0.1 ETH cost of MIN_FALLBACK_RESERVE. Thus, overpaying when it is not necessary.

This happens due to the lack of considering the already existing deposted gas amount.

Note: for simplicity, we assumed that tx.gasPrice didn't change.


#### About _forceRevert
`_forceRevert` withdraws all execution budget.
```
	// Withdraw all execution gas budget from anycall for tx to revert with "no enough budget"
	if (executionBudget > 0) try anycallConfig.withdraw(executionBudget) {} catch {}
```

So Anycall Executor will revert if there is not enough budget. This is done at 
```solidity
	uint256 budget = executionBudget[_from];
	require(budget > totalCost, "no enough budget");
	executionBudget[_from] = budget - totalCost;
```
https://github.com/anyswap/multichain-smart-contracts/blob/main/contracts/anycall/v7/AnycallV7Config.sol#L206C42-L206C58

This way we avoid reverting directly. Instead, we let Anycall Executor to revert avoiding triggering the fallback.

## Tools Used
Manual analysis

## Recommended Mitigation Steps


For  `retryDeposit`, use the internal function `_requiresFallbackGas(uint256 _depositedGas)` instead of the modifier. Pass the existing deposited gas + msg.value to the function.
Example:
```solidity
_requiresFallbackGas(getDeposit[_depositNonce].depositedGas+msg.value)
```

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1415-L1417


## Assessed type

Other