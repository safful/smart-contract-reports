## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-16

# [Overpaying remaining gas to the user or failing anyExecute call due to incorrect gas unit calculation in BranchBridgeAgent](https://github.com/code-423n4/2023-05-maia-findings/issues/607) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1018-L1054


# Vulnerability details

## Impact

#### Context:
`anyExecute` method is called by Anycall Executor on the destination chain to execute interaction. The user has to pay for the remote call execution gas, this is done at the end of the call. However, if there is not enough gasRemaining, the  `anyExecute` will be reverted due to a revert caused by the Anycall Executor.

Here is the calculation for the gas used
```solidity
	///Save gas left
	uint256 gasLeft = gasleft();

	//Get Branch Environment Execution Cost
	uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

	//Check if sufficient balance
	if (minExecCost > gasRemaining) {
		_forceRevert();
		return;
	}
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1018-L1054

`_forceRevert` will withdraw all execution budget.
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

#### (1) Gas Calculation:

To calculate how much the user has to pay, the following formula is used:

```solidity
        //Get Branch Environment Execution Cost
        uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);
```

Gas units are calculated as follows:
- Store gasleft() at initialGas at the beginning of `anyExecute` method
```solidity
	//Get Initial Gas Checkpoint
	uint256 initialGas = gasleft();
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1125

- Nearly at the end of the method, deduct gasleft() from initialGas. This covers everything between initial gas checkpoint and end gas checkpoint.
```solidity
        ///Save gas left
        uint256 gasLeft = gasleft();

        //Get Branch Environment Execution Cost
        uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);
```

- Add MIN_EXECUTION_OVERHEAD which is 160_000.  
```
    uint256 internal constant MIN_EXECUTION_OVERHEAD = 160_000; // 100_000 for anycall + 35_000 Pre 1st Gas Checkpoint Execution + 25_000 Post last Gas Checkpoint Executions
```

This overhead is supposed to cover:
- **100_000 for anycall**. This is extra cost required by Anycall
```solidity
Line:38	
uint256 constant EXECUTION_OVERHEAD = 100000;
	.
	.
Line:203	
uint256 gasUsed = _prevGasLeft + EXECUTION_OVERHEAD - gasleft();
```
https://github.com/anyswap/multichain-smart-contracts/blob/main/contracts/anycall/v7/AnycallV7Config.sol#L203

- **35_000 Pre 1st Gas Checkpoint Execution**. For example, to cover the modifier `requiresExecutor`

- **25_000 Post Last Gas Checkpoint Execution**. To cover everthing after the end gas checkpoint.
```solidity
	//Get Branch Environment Execution Cost
	uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

	//Check if sufficient balance
	if (minExecCost > gasRemaining) {
		_forceRevert();
		return;
	}

	//Replenish Gas
	_replenishGas(minExecCost);

	//Transfer gas remaining to recipient
	SafeTransferLib.safeTransferETH(_recipient, gasRemaining - minExecCost);

	//Save Gas
	uint256 gasAfterTransfer = gasleft();

	//Check if sufficient balance
	if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD) {
		_forceRevert();
		return;
	}
```


The issue is that **60_000 is not enough to cover pre 1st gas checkpoint and post last gas checkpoint**. This means that the user paying less than the actual gas cost. According to the sponsor, Bridge Agent deployer deposits first time into anycallConfig where the goal is to replenish the execution budget after use every time. The issue could possibly lead to:
1. **Overpaying the remaining gas the user **.
2. **execution budget is decreasing over time (slow draining)** in case it has funds already.
3. **anyExecute calls will fail** since the calculation of the gas used in the Anycall contracts is way bigger. In Anycall, this is done by the modifier `chargeDestFee`
	- modifier `chargeDestFee`
	```solidity
		modifier chargeDestFee(address _from, uint256 _flags) {
		if (_isSet(_flags, AnycallFlags.FLAG_PAY_FEE_ON_DEST)) {
			uint256 _prevGasLeft = gasleft();
			_;
			IAnycallConfig(config).chargeFeeOnDestChain(_from, _prevGasLeft);
		} else {
			_;
		}
	}
	```
	https://github.com/anyswap/multichain-smart-contracts/blob/main/contracts/anycall/v7/AnycallV7Upgradeable.sol#L163-L171
	
	- function `chargeFeeOnDestChain`
	```solidity
		function chargeFeeOnDestChain(address _from, uint256 _prevGasLeft)
			external
			onlyAnycallContract
		{
			if (!_isSet(mode, FREE_MODE)) {
				uint256 gasUsed = _prevGasLeft + EXECUTION_OVERHEAD - gasleft();
				uint256 totalCost = gasUsed * (tx.gasprice + _feeData.premium);
				uint256 budget = executionBudget[_from];
				require(budget > totalCost, "no enough budget");
				executionBudget[_from] = budget - totalCost;
				_feeData.accruedFees += uint128(totalCost);
			}
		}
	```
	https://github.com/anyswap/multichain-smart-contracts/blob/main/contracts/anycall/v7/AnycallV7Config.sol#L203

#### (3) Gas Calculation in AnyCall:

There is also a gas consumption at `anyExec` method called by the MPC (in AnyCall) here
```solidity
    function anyExec(
        address _to,
        bytes calldata _data,
        string calldata _appID,
        RequestContext calldata _ctx,
        bytes calldata _extdata
    )
        external
        virtual
        lock
        whenNotPaused
        chargeDestFee(_to, _ctx.flags) // <= starting from here
        onlyMPC
    {
		.
		.
		.
		bool success = _execute(_to, _data, _ctx, _extdata);
		.
		.
   }
```

https://github.com/anyswap/multichain-smart-contracts/blob/main/contracts/anycall/v7/AnycallV7Upgradeable.sol#L276

**The gas is nearly 110_000**. It is not taken into account. 


#### (3) Base Fee & Input Data Fee:

From [Ethereum yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf):

> Gtransaction 21000 Paid for every transaction
> Gtxdatazero 4 Paid for every zero byte of data or code for a transaction.
> Gtxdatanonzero 16 Paid for every non-zero byte of data or code for a transaction.

So
1. We have 21_000 as a base fee. This should be taken into account. However, it is paid by AnyCall, since the TX is sent by MPC. So, we are fine here. Probably this explains the overhead (100_000) added by anycall

2. Because `anyExecute` method has bytes data to be passed, we have extra gas consumption which is not taken into account. 
For every zero byte => 4 
For every non-zero byte => 16

So generally speaking, the bigger the data is, the bigger the gas becomes. you can simply prove this by adding arbitrary data to `anyExecute` method in PoC#1 test below. and you will see the gas spent increases.


#### Summary

1. **MIN_EXECUTION_OVERHEAD is underestimated**.
2. **The gas consumed by `anyExec` method called by the MPC is not considered**.
3. **Input data fee isn't taken into account**.

There are two PoCs proving the first two points above. The third point can be proven by simply adding arbitrary data to `anyExecute` method in PoC#1 test.

## Proof of Concept

### PoC#1 (MIN_EXECUTION_OVERHEAD is underestimated)

#### Overview
This PoC is independent from the codebase (but uses the same code).
There are two contracts simulating `BranchBridgeAgent.anyExecute`.
1. **BranchBridgeAgent** which has the code of pre 1st gas checkpoint and  post last gas checkpoint.
2. **BranchBridgeAgentEmpty** which has the code of pre 1st gas checkpoint and  post last gas checkpoint **commented out**.

We run the same test for both, the difference in gas is what’s at least nearly the minimum required to cover pre 1st gas checkpoint and  post last gas checkpoint.
In this case here it is **78097** which is bigger than 60_000.

Here is the output of the test:

```sh
[PASS] test_calcgas() (gas: 119050)
Logs:
  branchBridgeAgent.anyExecute Gas Spent => 92852

[PASS] test_calcgasEmpty() (gas: 44461)
Logs:
  branchBridgeAgentEmpty.anyExecute Gas Spent => 14755
```

**92852-14755 = 78097**

#### Explanation 

`BranchBridgeAgent.anyExecute` method depends on the following external calls:
1. `AnycallExecutor.context()`
2. `AnycallProxy.config()`
3. `AnycallConfig.executionBudget()`
4. `AnycallConfig.withdraw()`
5. `AnycallConfig.deposit()`
6. `WETH9.withdraw()`

For this reason, I've copied the same code from [multichain-smart-contracts](https://github.com/anyswap/multichain-smart-contracts). For WETH9, I've used the contract from the codebase which has minimal code.

Please note that:
- **tx.gasprice** is replaced with a fixed value in `_payExecutionGas` method as it is not available in Foundry.
- In `_replenishGas`, reading the config via `IAnycallProxy(localAnyCallAddress).config()` is replaced with Immediate call for simplicity. In other words, avoiding proxy to make the PoC simpler and shorter. However, if done with proxy the gas used would increase. So in both ways, it is in favor of the PoC.
- The condition `if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD)` is replaced with `if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD && false)` . This is to avoid entering the forceRevert. The increase of gas here is negligible anyway. 

####  The coded PoC 
- Foundry.toml
```sh
	[profile.default]
	solc = '0.8.17'
	src = 'solidity'
	test = 'solidity/test'
	out = 'out'
	libs = ['lib']
	fuzz_runs = 1000
	optimizer_runs = 10_000
```

- .gitmodules
```sh
	[submodule "lib/ds-test"]
		path = lib/ds-test
		url = https://github.com/dapphub/ds-test
		branch = master
	[submodule "lib/forge-std"]
		path = lib/forge-std
		url = https://github.com/brockelmore/forge-std
		branch = master
```

- remappings.txt
```sh
	ds-test/=lib/ds-test/src
	forge-std/=lib/forge-std/src
```

- Test File
```solidity
// PoC => Maia OmniChain: gasCalculation in BranchBridgeAgent  
pragma solidity >=0.8.4 <0.9.0;

import {Test} from "forge-std/Test.sol";
import "forge-std/console.sol";

import {DSTest} from "ds-test/test.sol";



library SafeTransferLib {
    /*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
    /*                       CUSTOM ERRORS                        */
    /*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/

    /// @dev The ETH transfer has failed.
    error ETHTransferFailed();

    /// @dev The ERC20 `transferFrom` has failed.
    error TransferFromFailed();

    /// @dev The ERC20 `transfer` has failed.
    error TransferFailed();

    /// @dev The ERC20 `approve` has failed.
    error ApproveFailed();

    /*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
    /*                         CONSTANTS                          */
    /*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/

    /// @dev Suggested gas stipend for contract receiving ETH
    /// that disallows any storage writes.
    uint256 internal constant _GAS_STIPEND_NO_STORAGE_WRITES = 2300;

    /// @dev Suggested gas stipend for contract receiving ETH to perform a few
    /// storage reads and writes, but low enough to prevent griefing.
    /// Multiply by a small constant (e.g. 2), if needed.
    uint256 internal constant _GAS_STIPEND_NO_GRIEF = 100000;

    /*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
    /*                       ETH OPERATIONS                       */
    /*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/

    /// @dev Sends `amount` (in wei) ETH to `to`.
    /// Reverts upon failure.
    ///
    /// Note: This implementation does NOT protect against gas griefing.
    /// Please use `forceSafeTransferETH` for gas griefing protection.
    function safeTransferETH(address to, uint256 amount) internal {
        /// @solidity memory-safe-assembly
        assembly {
            // Transfer the ETH and check if it succeeded or not.
            if iszero(call(gas(), to, amount, 0, 0, 0, 0)) {
                // Store the function selector of `ETHTransferFailed()`.
                mstore(0x00, 0xb12d13eb)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
        }
    }

    /// @dev Force sends `amount` (in wei) ETH to `to`, with a `gasStipend`.
    /// The `gasStipend` can be set to a low enough value to prevent
    /// storage writes or gas griefing.
    ///
    /// If sending via the normal procedure fails, force sends the ETH by
    /// creating a temporary contract which uses `SELFDESTRUCT` to force send the ETH.
    ///
    /// Reverts if the current contract has insufficient balance.
    function forceSafeTransferETH(address to, uint256 amount, uint256 gasStipend) internal {
        /// @solidity memory-safe-assembly
        assembly {
            // If insufficient balance, revert.
            if lt(selfbalance(), amount) {
                // Store the function selector of `ETHTransferFailed()`.
                mstore(0x00, 0xb12d13eb)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
            // Transfer the ETH and check if it succeeded or not.
            if iszero(call(gasStipend, to, amount, 0, 0, 0, 0)) {
                mstore(0x00, to) // Store the address in scratch space.
                mstore8(0x0b, 0x73) // Opcode `PUSH20`.
                mstore8(0x20, 0xff) // Opcode `SELFDESTRUCT`.
                // We can directly use `SELFDESTRUCT` in the contract creation.
                // Compatible with `SENDALL`: https://eips.ethereum.org/EIPS/eip-4758
                if iszero(create(amount, 0x0b, 0x16)) {
                    // To coerce gas estimation to provide enough gas for the `create` above.
                    if iszero(gt(gas(), 1000000)) { revert(0, 0) }
                }
            }
        }
    }

    /// @dev Force sends `amount` (in wei) ETH to `to`, with a gas stipend
    /// equal to `_GAS_STIPEND_NO_GRIEF`. This gas stipend is a reasonable default
    /// for 99% of cases and can be overridden with the three-argument version of this
    /// function if necessary.
    ///
    /// If sending via the normal procedure fails, force sends the ETH by
    /// creating a temporary contract which uses `SELFDESTRUCT` to force send the ETH.
    ///
    /// Reverts if the current contract has insufficient balance.
    function forceSafeTransferETH(address to, uint256 amount) internal {
        // Manually inlined because the compiler doesn't inline functions with branches.
        /// @solidity memory-safe-assembly
        assembly {
            // If insufficient balance, revert.
            if lt(selfbalance(), amount) {
                // Store the function selector of `ETHTransferFailed()`.
                mstore(0x00, 0xb12d13eb)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
            // Transfer the ETH and check if it succeeded or not.
            if iszero(call(_GAS_STIPEND_NO_GRIEF, to, amount, 0, 0, 0, 0)) {
                mstore(0x00, to) // Store the address in scratch space.
                mstore8(0x0b, 0x73) // Opcode `PUSH20`.
                mstore8(0x20, 0xff) // Opcode `SELFDESTRUCT`.
                // We can directly use `SELFDESTRUCT` in the contract creation.
                // Compatible with `SENDALL`: https://eips.ethereum.org/EIPS/eip-4758
                if iszero(create(amount, 0x0b, 0x16)) {
                    // To coerce gas estimation to provide enough gas for the `create` above.
                    if iszero(gt(gas(), 1000000)) { revert(0, 0) }
                }
            }
        }
    }

    /// @dev Sends `amount` (in wei) ETH to `to`, with a `gasStipend`.
    /// The `gasStipend` can be set to a low enough value to prevent
    /// storage writes or gas griefing.
    ///
    /// Simply use `gasleft()` for `gasStipend` if you don't need a gas stipend.
    ///
    /// Note: Does NOT revert upon failure.
    /// Returns whether the transfer of ETH is successful instead.
    function trySafeTransferETH(address to, uint256 amount, uint256 gasStipend)
        internal
        returns (bool success)
    {
        /// @solidity memory-safe-assembly
        assembly {
            // Transfer the ETH and check if it succeeded or not.
            success := call(gasStipend, to, amount, 0, 0, 0, 0)
        }
    }

    /*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/
    /*                      ERC20 OPERATIONS                      */
    /*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/

    /// @dev Sends `amount` of ERC20 `token` from `from` to `to`.
    /// Reverts upon failure.
    ///
    /// The `from` account must have at least `amount` approved for
    /// the current contract to manage.
    function safeTransferFrom(address token, address from, address to, uint256 amount) internal {
        /// @solidity memory-safe-assembly
        assembly {
            let m := mload(0x40) // Cache the free memory pointer.

            mstore(0x60, amount) // Store the `amount` argument.
            mstore(0x40, to) // Store the `to` argument.
            mstore(0x2c, shl(96, from)) // Store the `from` argument.
            // Store the function selector of `transferFrom(address,address,uint256)`.
            mstore(0x0c, 0x23b872dd000000000000000000000000)

            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    // Set success to whether the call reverted, if not we check it either
                    // returned exactly 1 (can't just be non-zero data), or had no return data.
                    or(eq(mload(0x00), 1), iszero(returndatasize())),
                    call(gas(), token, 0, 0x1c, 0x64, 0x00, 0x20)
                )
            ) {
                // Store the function selector of `TransferFromFailed()`.
                mstore(0x00, 0x7939f424)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }

            mstore(0x60, 0) // Restore the zero slot to zero.
            mstore(0x40, m) // Restore the free memory pointer.
        }
    }

    /// @dev Sends all of ERC20 `token` from `from` to `to`.
    /// Reverts upon failure.
    ///
    /// The `from` account must have their entire balance approved for
    /// the current contract to manage.
    function safeTransferAllFrom(address token, address from, address to)
        internal
        returns (uint256 amount)
    {
        /// @solidity memory-safe-assembly
        assembly {
            let m := mload(0x40) // Cache the free memory pointer.

            mstore(0x40, to) // Store the `to` argument.
            mstore(0x2c, shl(96, from)) // Store the `from` argument.
            // Store the function selector of `balanceOf(address)`.
            mstore(0x0c, 0x70a08231000000000000000000000000)
            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    gt(returndatasize(), 0x1f), // At least 32 bytes returned.
                    staticcall(gas(), token, 0x1c, 0x24, 0x60, 0x20)
                )
            ) {
                // Store the function selector of `TransferFromFailed()`.
                mstore(0x00, 0x7939f424)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }

            // Store the function selector of `transferFrom(address,address,uint256)`.
            mstore(0x00, 0x23b872dd)
            // The `amount` argument is already written to the memory word at 0x60.
            amount := mload(0x60)

            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    // Set success to whether the call reverted, if not we check it either
                    // returned exactly 1 (can't just be non-zero data), or had no return data.
                    or(eq(mload(0x00), 1), iszero(returndatasize())),
                    call(gas(), token, 0, 0x1c, 0x64, 0x00, 0x20)
                )
            ) {
                // Store the function selector of `TransferFromFailed()`.
                mstore(0x00, 0x7939f424)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }

            mstore(0x60, 0) // Restore the zero slot to zero.
            mstore(0x40, m) // Restore the free memory pointer.
        }
    }

    /// @dev Sends `amount` of ERC20 `token` from the current contract to `to`.
    /// Reverts upon failure.
    function safeTransfer(address token, address to, uint256 amount) internal {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x14, to) // Store the `to` argument.
            mstore(0x34, amount) // Store the `amount` argument.
            // Store the function selector of `transfer(address,uint256)`.
            mstore(0x00, 0xa9059cbb000000000000000000000000)

            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    // Set success to whether the call reverted, if not we check it either
                    // returned exactly 1 (can't just be non-zero data), or had no return data.
                    or(eq(mload(0x00), 1), iszero(returndatasize())),
                    call(gas(), token, 0, 0x10, 0x44, 0x00, 0x20)
                )
            ) {
                // Store the function selector of `TransferFailed()`.
                mstore(0x00, 0x90b8ec18)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
            // Restore the part of the free memory pointer that was overwritten.
            mstore(0x34, 0)
        }
    }

    /// @dev Sends all of ERC20 `token` from the current contract to `to`.
    /// Reverts upon failure.
    function safeTransferAll(address token, address to) internal returns (uint256 amount) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x00, 0x70a08231) // Store the function selector of `balanceOf(address)`.
            mstore(0x20, address()) // Store the address of the current contract.
            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    gt(returndatasize(), 0x1f), // At least 32 bytes returned.
                    staticcall(gas(), token, 0x1c, 0x24, 0x34, 0x20)
                )
            ) {
                // Store the function selector of `TransferFailed()`.
                mstore(0x00, 0x90b8ec18)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }

            mstore(0x14, to) // Store the `to` argument.
            // The `amount` argument is already written to the memory word at 0x34.
            amount := mload(0x34)
            // Store the function selector of `transfer(address,uint256)`.
            mstore(0x00, 0xa9059cbb000000000000000000000000)

            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    // Set success to whether the call reverted, if not we check it either
                    // returned exactly 1 (can't just be non-zero data), or had no return data.
                    or(eq(mload(0x00), 1), iszero(returndatasize())),
                    call(gas(), token, 0, 0x10, 0x44, 0x00, 0x20)
                )
            ) {
                // Store the function selector of `TransferFailed()`.
                mstore(0x00, 0x90b8ec18)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
            // Restore the part of the free memory pointer that was overwritten.
            mstore(0x34, 0)
        }
    }

    /// @dev Sets `amount` of ERC20 `token` for `to` to manage on behalf of the current contract.
    /// Reverts upon failure.
    function safeApprove(address token, address to, uint256 amount) internal {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x14, to) // Store the `to` argument.
            mstore(0x34, amount) // Store the `amount` argument.
            // Store the function selector of `approve(address,uint256)`.
            mstore(0x00, 0x095ea7b3000000000000000000000000)

            if iszero(
                and( // The arguments of `and` are evaluated from right to left.
                    // Set success to whether the call reverted, if not we check it either
                    // returned exactly 1 (can't just be non-zero data), or had no return data.
                    or(eq(mload(0x00), 1), iszero(returndatasize())),
                    call(gas(), token, 0, 0x10, 0x44, 0x00, 0x20)
                )
            ) {
                // Store the function selector of `ApproveFailed()`.
                mstore(0x00, 0x3e3f8f73)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
            // Restore the part of the free memory pointer that was overwritten.
            mstore(0x34, 0)
        }
    }

    /// @dev Returns the amount of ERC20 `token` owned by `account`.
    /// Returns zero if the `token` does not exist.
    function balanceOf(address token, address account) internal view returns (uint256 amount) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x14, account) // Store the `account` argument.
            // Store the function selector of `balanceOf(address)`.
            mstore(0x00, 0x70a08231000000000000000000000000)
            amount :=
                mul(
                    mload(0x20),
                    and( // The arguments of `and` are evaluated from right to left.
                        gt(returndatasize(), 0x1f), // At least 32 bytes returned.
                        staticcall(gas(), token, 0x10, 0x24, 0x20, 0x20)
                    )
                )
        }
    }
}

interface IAnycallExecutor {
    function context()
        external
        view
        returns (address from, uint256 fromChainID, uint256 nonce);

    function execute(
        address _to,
        bytes calldata _data,
        address _from,
        uint256 _fromChainID,
        uint256 _nonce,
        uint256 _flags,
        bytes calldata _extdata
    ) external returns (bool success, bytes memory result);
}

interface IAnycallConfig {
    function calcSrcFees(
        address _app,
        uint256 _toChainID,
        uint256 _dataLength
    ) external view returns (uint256);

    function executionBudget(address _app) external view returns (uint256);

    function deposit(address _account) external payable;

    function withdraw(uint256 _amount) external;
}

interface IAnycallProxy {
    function executor() external view returns (address);

    function config() external view returns (address);

    function anyCall(
        address _to,
        bytes calldata _data,
        uint256 _toChainID,
        uint256 _flags,
        bytes calldata _extdata
    ) external payable;

    function anyCall(
        string calldata _to,
        bytes calldata _data,
        uint256 _toChainID,
        uint256 _flags,
        bytes calldata _extdata
    ) external payable;
}

contract WETH9 {
    string public name = "Wrapped Ether";
    string public symbol = "WETH";
    uint8 public decimals = 18;

    event Approval(address indexed src, address indexed guy, uint256 wad);
    event Transfer(address indexed src, address indexed dst, uint256 wad);
    event Deposit(address indexed dst, uint256 wad);
    event Withdrawal(address indexed src, uint256 wad);

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // function receive() external payable {
    //   deposit();
    // }

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }

    function withdraw(uint256 wad) public {
        require(balanceOf[msg.sender] >= wad);
        balanceOf[msg.sender] -= wad;
        payable(msg.sender).transfer(wad);
        emit Withdrawal(msg.sender, wad);
    }

    function totalSupply() public view returns (uint256) {
        return address(this).balance;
    }

    function approve(address guy, uint256 wad) public returns (bool) {
        allowance[msg.sender][guy] = wad;
        emit Approval(msg.sender, guy, wad);
        return true;
    }

    function transfer(address dst, uint256 wad) public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }

    function transferFrom(
        address src,
        address dst,
        uint256 wad
    ) public returns (bool) {
        require(balanceOf[src] >= wad);

        if (src != msg.sender && allowance[src][msg.sender] != 255) {
            require(allowance[src][msg.sender] >= wad);
            allowance[src][msg.sender] -= wad;
        }

        balanceOf[src] -= wad;
        balanceOf[dst] += wad;

        emit Transfer(src, dst, wad);

        return true;
    }
}

contract AnycallExecutor {
    struct Context {
        address from;
        uint256 fromChainID;
        uint256 nonce;
    }
    // Context public override context;
    Context public context;

    constructor() {
        context.fromChainID = 1;
        context.from = address(2);
        context.nonce = 1;
    }
}

contract AnycallV7Config {
    event Deposit(address indexed account, uint256 amount);

    mapping(address => uint256) public executionBudget;

    /// @notice Deposit native currency crediting `_account` for execution costs on this chain
    /// @param _account The account to deposit and credit for
    function deposit(address _account) external payable {
        executionBudget[_account] += msg.value;
        emit Deposit(_account, msg.value);
    }
}

contract BranchBridgeAgent {
    error AnycallUnauthorizedCaller();
    error GasErrorOrRepeatedTx();

    uint256 public remoteCallDepositedGas;

    uint256 internal constant MIN_EXECUTION_OVERHEAD = 160_000; // 100_000 for anycall + 35_000 Pre 1st Gas Checkpoint Execution + 25_000 Post last Gas Checkpoint Executions
    uint256 internal constant TRANSFER_OVERHEAD = 24_000;


    WETH9 public immutable wrappedNativeToken;

    AnycallV7Config public anycallV7Config;

    uint256 public accumulatedFees;

    /// @notice Local Chain Id
    uint24 public immutable localChainId;

    /// @notice Address for Bridge Agent who processes requests submitted for the Root Router Address where cross-chain requests are executed in the Root Chain.
    address public immutable rootBridgeAgentAddress;
    /// @notice Local Anyexec Address
    address public immutable localAnyCallExecutorAddress;

    /// @notice Address for Local AnycallV7 Proxy Address where cross-chain requests are sent to the Root Chain Router.
    address public immutable localAnyCallAddress;

    constructor() {
        AnycallExecutor anycallExecutor = new AnycallExecutor();
        localAnyCallExecutorAddress = address(anycallExecutor);

        localChainId = 1;

        wrappedNativeToken = new WETH9();

        localAnyCallAddress = address(3);

        rootBridgeAgentAddress = address(2);

        anycallV7Config = new AnycallV7Config();
    }

    modifier requiresExecutor() {
        _requiresExecutor();
        _;
    }

    function _requiresExecutor() internal view {
                if (msg.sender != localAnyCallExecutorAddress) revert AnycallUnauthorizedCaller();
        (address from,,) = IAnycallExecutor(localAnyCallExecutorAddress).context();
        if (from != rootBridgeAgentAddress) revert AnycallUnauthorizedCaller();

    }

    function _replenishGas(uint256 _executionGasSpent) internal virtual {
        //Deposit Gas
        anycallV7Config.deposit{value: _executionGasSpent}(address(this));
        // IAnycallConfig(IAnycallProxy(localAnyCallAddress).config()).deposit{value: _executionGasSpent}(address(this));
    }

    function _forceRevert() internal virtual {
        IAnycallConfig anycallConfig = IAnycallConfig(IAnycallProxy(localAnyCallAddress).config());
        uint256 executionBudget = anycallConfig.executionBudget(address(this));

        // Withdraw all execution gas budget from anycall for tx to revert with "no enough budget"
        if (executionBudget > 0) try anycallConfig.withdraw(executionBudget) {} catch {}
    }

   /**
     * @notice Internal function repays gas used by Branch Bridge Agent to fulfill remote initiated interaction.
     *   @param _recipient address to send excess gas to.
     *   @param _initialGas gas used by Branch Bridge Agent.
     */
    function _payExecutionGas(address _recipient, uint256 _initialGas) internal virtual {
        //Gas remaining
        uint256 gasRemaining = wrappedNativeToken.balanceOf(address(this));

        //Unwrap Gas
        wrappedNativeToken.withdraw(gasRemaining);

        //Delete Remote Initiated Action State
        delete(remoteCallDepositedGas);

        ///Save gas left
        uint256 gasLeft = gasleft();

        //Get Branch Environment Execution Cost
        // Assume tx.gasPrice 1e9 
        uint256 minExecCost = 1e9 * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

        //Check if sufficient balance
        if (minExecCost > gasRemaining) {
            _forceRevert();
            return;
        }

        //Replenish Gas
        _replenishGas(minExecCost);

        //Transfer gas remaining to recipient
        SafeTransferLib.safeTransferETH(_recipient, gasRemaining - minExecCost);

        //Save Gas
        uint256 gasAfterTransfer = gasleft();

        //Check if sufficient balance
        if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD && false) { // added false here so it doesn't enter.  
            _forceRevert();
            return;
        }
    }

    function anyExecute(
        bytes memory data
    )
        public
        virtual
        requiresExecutor
        returns (bool success, bytes memory result)
    {
        //Get Initial Gas Checkpoint
        uint256 initialGas = gasleft();

        //Action Recipient
        address recipient = address(0x1); // for simplicity and since it is irrelevant //address(uint160(bytes20(data[PARAMS_START:PARAMS_START_SIGNED])));


        // Other Code Here


        //Deduct gas costs from deposit and replenish this bridge agent's execution budget.
        _payExecutionGas(recipient, initialGas);
    }

    function depositIntoWeth(uint256 amt) external {
        wrappedNativeToken.deposit{value: amt}();
    }

    fallback() external payable {}
}


contract BranchBridgeAgentEmpty {
    error AnycallUnauthorizedCaller();
    error GasErrorOrRepeatedTx();

    uint256 public remoteCallDepositedGas;

    uint256 internal constant MIN_EXECUTION_OVERHEAD = 160_000; // 100_000 for anycall + 35_000 Pre 1st Gas Checkpoint Execution + 25_000 Post last Gas Checkpoint Executions
    uint256 internal constant TRANSFER_OVERHEAD = 24_000;


    WETH9 public immutable wrappedNativeToken;

    AnycallV7Config public anycallV7Config;

    uint256 public accumulatedFees;

    /// @notice Local Chain Id
    uint24 public immutable localChainId;

    /// @notice Address for Bridge Agent who processes requests submitted for the Root Router Address where cross-chain requests are executed in the Root Chain.
    address public immutable rootBridgeAgentAddress;
    /// @notice Local Anyexec Address
    address public immutable localAnyCallExecutorAddress;

    /// @notice Address for Local AnycallV7 Proxy Address where cross-chain requests are sent to the Root Chain Router.
    address public immutable localAnyCallAddress;

    constructor() {
        AnycallExecutor anycallExecutor = new AnycallExecutor();
        localAnyCallExecutorAddress = address(anycallExecutor);

        localChainId = 1;

        wrappedNativeToken = new WETH9();

        localAnyCallAddress = address(3);

        rootBridgeAgentAddress = address(2);

        anycallV7Config = new AnycallV7Config();
    }

    modifier requiresExecutor() {
        _requiresExecutor();
        _;
    }

    function _requiresExecutor() internal view {
                if (msg.sender != localAnyCallExecutorAddress) revert AnycallUnauthorizedCaller();
        (address from,,) = IAnycallExecutor(localAnyCallExecutorAddress).context();
        if (from != rootBridgeAgentAddress) revert AnycallUnauthorizedCaller();

    }

    function _replenishGas(uint256 _executionGasSpent) internal virtual {
        //Deposit Gas
        anycallV7Config.deposit{value: _executionGasSpent}(address(this));
        // IAnycallConfig(IAnycallProxy(localAnyCallAddress).config()).deposit{value: _executionGasSpent}(address(this));
    }

    function _forceRevert() internal virtual {
        IAnycallConfig anycallConfig = IAnycallConfig(IAnycallProxy(localAnyCallAddress).config());
        uint256 executionBudget = anycallConfig.executionBudget(address(this));

        // Withdraw all execution gas budget from anycall for tx to revert with "no enough budget"
        if (executionBudget > 0) try anycallConfig.withdraw(executionBudget) {} catch {}
    }

   /**
     * @notice Internal function repays gas used by Branch Bridge Agent to fulfill remote initiated interaction.
     *   @param _recipient address to send excess gas to.
     *   @param _initialGas gas used by Branch Bridge Agent.
     */
    function _payExecutionGas(address _recipient, uint256 _initialGas) internal virtual {
        //Gas remaining
        uint256 gasRemaining = wrappedNativeToken.balanceOf(address(this));

        //Unwrap Gas
        wrappedNativeToken.withdraw(gasRemaining);

        //Delete Remote Initiated Action State
        delete(remoteCallDepositedGas);

        ///Save gas left
        uint256 gasLeft = gasleft(); // Everything after this is not taken into account

        //Get Branch Environment Execution Cost
        // Assume tx.gasPrice 1e9 
        // uint256 minExecCost = 1e9 * (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

        // //Check if sufficient balance
        // if (minExecCost > gasRemaining) {
        //     _forceRevert();
        //     return;
        // }

        // //Replenish Gas
        // _replenishGas(minExecCost);

        // //Transfer gas remaining to recipient
        // SafeTransferLib.safeTransferETH(_recipient, gasRemaining - minExecCost);

        // //Save Gas
        // uint256 gasAfterTransfer = gasleft();

        // //Check if sufficient balance
        // if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD && false) { // added false here so it doesn't enter.  
        //     _forceRevert();
        //     return;
        // }
    }

    function anyExecute(
        bytes memory data
    )
        public
        virtual
        // requiresExecutor
        returns (bool success, bytes memory result)
    {
        //Get Initial Gas Checkpoint
        uint256 initialGas = gasleft();

        //Action Recipient
        address recipient = address(0x1); // for simplicity and since it is irrelevant //address(uint160(bytes20(data[PARAMS_START:PARAMS_START_SIGNED])));


        // Other Code Here


        //Deduct gas costs from deposit and replenish this bridge agent's execution budget.
        _payExecutionGas(recipient, initialGas);
    }

    function depositIntoWeth(uint256 amt) external {
        wrappedNativeToken.deposit{value: amt}();
    }

    fallback() external payable {}
}


contract GasCalc is DSTest, Test {
    BranchBridgeAgent branchBridgeAgent;
    BranchBridgeAgentEmpty branchBridgeAgentEmpty;

    function setUp() public {
        branchBridgeAgentEmpty = new BranchBridgeAgentEmpty();
        vm.deal(address(branchBridgeAgentEmpty.localAnyCallExecutorAddress()), 100 ether); // executer pays gas
        vm.deal(address(branchBridgeAgentEmpty), 100 ether);


        branchBridgeAgent = new BranchBridgeAgent();
        vm.deal(address(branchBridgeAgent.localAnyCallExecutorAddress()), 100 ether); // executer pays gas
        vm.deal(address(branchBridgeAgent), 100 ether);


    }

    // code after end checkpoint gasLeft not included
    function test_calcgasEmpty() public {
 // add weth balance
        branchBridgeAgentEmpty.depositIntoWeth(100 ether);

        vm.prank(address(branchBridgeAgentEmpty.localAnyCallExecutorAddress()));
        uint256 gasStart_ = gasleft();
        branchBridgeAgentEmpty.anyExecute(bytes(""));
        uint256 gasEnd_ = gasleft();
        vm.stopPrank();
        uint256 gasSpent_ = gasStart_ - gasEnd_;
        console.log("branchBridgeAgentEmpty.anyExecute Gas Spent => %d", gasSpent_);
        


    }

    // code after end checkpoint gasLeft included
     function test_calcgas() public {

        // add weth balance
        branchBridgeAgent.depositIntoWeth(100 ether);

        vm.prank(address(branchBridgeAgent.localAnyCallExecutorAddress()));
        uint256 gasStart = gasleft();
        branchBridgeAgent.anyExecute(bytes(""));
        uint256 gasEnd = gasleft();
        vm.stopPrank();
        uint256 gasSpent = gasStart - gasEnd;
        console.log("branchBridgeAgent.anyExecute Gas Spent => %d", gasSpent);


    }
}


```


### PoC#2 (The gas consumed by `anyExec` method in AnyCall)

#### Overview

We have contracts that simulating Anycall contracts:
1. AnycallV7Config
2. AnycallExecutor
3. AnycallV7

The flow like this:
MPC => AnycallV7 => AnycallExecutor => IApp

In the code, `IApp(_to).anyExecute`  is commented out because we don't want to calculate its gas since it is done in PoC#1.

Here is the output of the test:

```sh
[PASS] test_gasInanycallv7() (gas: 102613)
Logs:
  anycallV7.anyExec Gas Spent => 110893
```

#### coded PoC
```solidity
// PoC => Maia OmniChain: gasCalculation in AnyCall v7  contracts
pragma solidity >=0.8.4 <0.9.0;

import {Test} from "forge-std/Test.sol";
import "forge-std/console.sol";

import {DSTest} from "ds-test/test.sol";

/// IAnycallConfig interface of the anycall config
interface IAnycallConfig {
    function checkCall(
        address _sender,
        bytes calldata _data,
        uint256 _toChainID,
        uint256 _flags
    ) external view returns (string memory _appID, uint256 _srcFees);

    function checkExec(
        string calldata _appID,
        address _from,
        address _to
    ) external view;

    function chargeFeeOnDestChain(address _from, uint256 _prevGasLeft) external;
}

/// IAnycallExecutor interface of the anycall executor
interface IAnycallExecutor {
    function context()
        external
        view
        returns (address from, uint256 fromChainID, uint256 nonce);

    function execute(
        address _to,
        bytes calldata _data,
        address _from,
        uint256 _fromChainID,
        uint256 _nonce,
        uint256 _flags,
        bytes calldata _extdata
    ) external returns (bool success, bytes memory result);
}

/// IApp interface of the application
interface IApp {
    /// (required) call on the destination chain to exec the interaction
    function anyExecute(bytes calldata _data)
        external
        returns (bool success, bytes memory result);

    /// (optional,advised) call back on the originating chain if the cross chain interaction fails
    /// `_data` is the orignal interaction arguments exec on the destination chain
    function anyFallback(bytes calldata _data)
        external
        returns (bool success, bytes memory result);
}

library AnycallFlags {
    // call flags which can be specified by user
    uint256 public constant FLAG_NONE = 0x0;
    uint256 public constant FLAG_MERGE_CONFIG_FLAGS = 0x1;
    uint256 public constant FLAG_PAY_FEE_ON_DEST = 0x1 << 1;
    uint256 public constant FLAG_ALLOW_FALLBACK = 0x1 << 2;

    // exec flags used internally
    uint256 public constant FLAG_EXEC_START_VALUE = 0x1 << 16;
    uint256 public constant FLAG_EXEC_FALLBACK = 0x1 << 16;
}

contract AnycallV7Config {
    uint256 public constant PERMISSIONLESS_MODE = 0x1;
    uint256 public constant FREE_MODE = 0x1 << 1;

    mapping(string => mapping(address => bool)) public appExecWhitelist;
    mapping(string => bool) public appBlacklist;

    uint256 public mode;

    uint256 public minReserveBudget;

    mapping(address => uint256) public executionBudget;

    constructor() {
         mode = PERMISSIONLESS_MODE;
    }

    function checkExec(
        string calldata _appID,
        address _from,
        address _to
    ) external view {
        require(!appBlacklist[_appID], "blacklist");

        if (!_isSet(mode, PERMISSIONLESS_MODE)) {
            require(appExecWhitelist[_appID][_to], "no permission");
        }

        if (!_isSet(mode, FREE_MODE)) {
            require(
                executionBudget[_from] >= minReserveBudget,
                "less than min budget"
            );
        }
    }

    function _isSet(
        uint256 _value,
        uint256 _testBits
    ) internal pure returns (bool) {
        return (_value & _testBits) == _testBits;
    }
}

contract AnycallExecutor {
    bytes32 public constant PAUSE_ALL_ROLE = 0x00;

    event Paused(bytes32 role);
    event Unpaused(bytes32 role);

    modifier whenNotPaused(bytes32 role) {
        require(
            !paused(role) && !paused(PAUSE_ALL_ROLE),
            "PausableControl: paused"
        );
        _;
    }
    mapping(bytes32 => bool) private _pausedRoles;
    mapping(address => bool) public isSupportedCaller;


    struct Context {
        address from;
        uint256 fromChainID;
        uint256 nonce;
    }
    // Context public override context;
    Context public context;

    function paused(bytes32 role) public view virtual returns (bool) {
        return _pausedRoles[role];
    }

    modifier onlyAuth() {
        require(isSupportedCaller[msg.sender], "not supported caller");
        _;
    }

    constructor(address anycall) {
        context.fromChainID = 1;
        context.from = address(2);
        context.nonce = 1;

        isSupportedCaller[anycall] = true;
    }

function _isSet(uint256 _value, uint256 _testBits)
        internal
        pure
        returns (bool)
    {
        return (_value & _testBits) == _testBits;
    }

    // @dev `_extdata` content is implementation based in each version
    function execute(
        address _to,
        bytes calldata _data,
        address _from,
        uint256 _fromChainID,
        uint256 _nonce,
        uint256 _flags,
        bytes calldata /*_extdata*/
    )
        external
        virtual
        onlyAuth
        whenNotPaused(PAUSE_ALL_ROLE)
        returns (bool success, bytes memory result)
    {
        bool isFallback = _isSet(_flags, AnycallFlags.FLAG_EXEC_FALLBACK);

        context = Context({
            from: _from,
            fromChainID: _fromChainID,
            nonce: _nonce
        });

        if (!isFallback) {
            // we skip calling anyExecute since it is irrelevant for this PoC
            // (success, result) = IApp(_to).anyExecute(_data);
        } else {
            (success, result) = IApp(_to).anyFallback(_data);
        }

        context = Context({from: address(0), fromChainID: 0, nonce: 0});
    }
}

contract AnycallV7 {

    event LogAnyCall(
        address indexed from,
        address to,
        bytes data,
        uint256 toChainID,
        uint256 flags,
        string appID,
        uint256 nonce,
        bytes extdata
    );
    event LogAnyCall(
        address indexed from,
        string to,
        bytes data,
        uint256 toChainID,
        uint256 flags,
        string appID,
        uint256 nonce,
        bytes extdata
    );

    event LogAnyExec(
        bytes32 indexed txhash,
        address indexed from,
        address indexed to,
        uint256 fromChainID,
        uint256 nonce,
        bool success,
        bytes result
    );


    event StoreRetryExecRecord(
        bytes32 indexed txhash,
        address indexed from,
        address indexed to,
        uint256 fromChainID,
        uint256 nonce,
        bytes data
    );

    // Context of the request on originating chain
    struct RequestContext {
        bytes32 txhash;
        address from;
        uint256 fromChainID;
        uint256 nonce;
        uint256 flags;
    }
    address public mpc;

    bool public paused;

    // applications should give permission to this executor
    address public executor;

    // anycall config contract
    address public config;

    mapping(bytes32 => bytes32) public retryExecRecords;
    bool public retryWithPermit;

    mapping(bytes32 => bool) public execCompleted;
    uint256 nonce;

    uint256 private unlocked;

    modifier lock() {
        require(unlocked == 1, "locked");
        unlocked = 0;
        _;
        unlocked = 1;
    }
    /// @dev Access control function
    modifier onlyMPC() {
        require(msg.sender == mpc, "only MPC");
        _;
    }

    /// @dev pausable control function
    modifier whenNotPaused() {
        require(!paused, "paused");
        _;
    }

    function _isSet(uint256 _value, uint256 _testBits)
        internal
        pure
        returns (bool)
    {
        return (_value & _testBits) == _testBits;
    }


    /// @dev Charge an account for execution costs on this chain
    /// @param _from The account to charge for execution costs
    modifier chargeDestFee(address _from, uint256 _flags) {
        if (_isSet(_flags, AnycallFlags.FLAG_PAY_FEE_ON_DEST)) {
            uint256 _prevGasLeft = gasleft();
            _;
            IAnycallConfig(config).chargeFeeOnDestChain(_from, _prevGasLeft);
        } else {
            _;
        }
    }

    constructor(address _mpc) {
        unlocked = 1; // needs to be unlocked initially
        mpc = _mpc;
        config = address(new AnycallV7Config());
        executor = address(new AnycallExecutor(address(this)));
    }

    /// @notice Calc unique ID
    function calcUniqID(
        bytes32 _txhash,
        address _from,
        uint256 _fromChainID,
        uint256 _nonce
    ) public pure returns (bytes32) {
        return keccak256(abi.encode(_txhash, _from, _fromChainID, _nonce));
    }

    function _execute(
        address _to,
        bytes memory _data,
        RequestContext memory _ctx,
        bytes memory _extdata
    ) internal returns (bool success) {
        bytes memory result;

        try
            IAnycallExecutor(executor).execute(
                _to,
                _data,
                _ctx.from,
                _ctx.fromChainID,
                _ctx.nonce,
                _ctx.flags,
                _extdata
            )
        returns (bool succ, bytes memory res) {
            (success, result) = (succ, res);
        } catch Error(string memory reason) {
            result = bytes(reason);
        } catch (bytes memory reason) {
            result = reason;
        }

        emit LogAnyExec(
            _ctx.txhash,
            _ctx.from,
            _to,
            _ctx.fromChainID,
            _ctx.nonce,
            success,
            result
        );
    }

    /**
        @notice Execute a cross chain interaction
        @dev Only callable by the MPC
        @param _to The cross chain interaction target
        @param _data The calldata supplied for interacting with target
        @param _appID The app identifier to check whitelist
        @param _ctx The context of the request on originating chain
        @param _extdata The extension data for execute context
    */
   // Note: changed from callback to memory so we can call it from the test contract
    function anyExec(
        address _to,
        bytes memory _data,
        string memory _appID,
        RequestContext memory _ctx, 
        bytes memory _extdata
    )
        external
        virtual
        lock
        whenNotPaused
        chargeDestFee(_to, _ctx.flags)
        onlyMPC
    {
        IAnycallConfig(config).checkExec(_appID, _ctx.from, _to);
        bytes32 uniqID = calcUniqID(
            _ctx.txhash,
            _ctx.from,
            _ctx.fromChainID,
            _ctx.nonce
        );
        require(!execCompleted[uniqID], "exec completed");

        bool success = _execute(_to, _data, _ctx, _extdata);
        // success = false on purpose, because when it is true, it consumes less gas. so we are considering worse case here

        // set exec completed (dont care success status)
        execCompleted[uniqID] = true;

        if (!success) {
            if (_isSet(_ctx.flags, AnycallFlags.FLAG_ALLOW_FALLBACK)) { 
                // this will be executed here since the call failed
                // Call the fallback on the originating chain
                nonce++;
                string memory appID = _appID; // fix Stack too deep
                emit LogAnyCall(
                    _to,
                    _ctx.from,
                    _data,
                    _ctx.fromChainID,
                    AnycallFlags.FLAG_EXEC_FALLBACK |
                        AnycallFlags.FLAG_PAY_FEE_ON_DEST, // pay fee on dest chain
                    appID,
                    nonce,
                    ""
                );
            } else {
                // Store retry record and emit a log
                bytes memory data = _data; // fix Stack too deep
                retryExecRecords[uniqID] = keccak256(abi.encode(_to, data));
                emit StoreRetryExecRecord(
                    _ctx.txhash,
                    _ctx.from,
                    _to,
                    _ctx.fromChainID,
                    _ctx.nonce,
                    data
                );
            }
        }
    }
}

contract GasCalcAnyCallv7 is DSTest, Test {


    AnycallV7 anycallV7;
    address mpc = vm.addr(7);
    function setUp() public {
        anycallV7 = new AnycallV7(mpc);
    }

    function test_gasInanycallv7() public {

        vm.prank(mpc);
        AnycallV7.RequestContext memory ctx = AnycallV7.RequestContext({
            txhash:keccak256(""),
            from:address(0),
            fromChainID:1,
            nonce:1,
            flags:AnycallFlags.FLAG_ALLOW_FALLBACK
        });
        uint256 gasStart_ = gasleft();
        anycallV7.anyExec(address(0),bytes(""),"1",ctx,bytes(""));
        uint256 gasEnd_ = gasleft();
        vm.stopPrank();
        uint256 gasSpent_ = gasStart_ - gasEnd_;
        console.log("anycallV7.anyExec Gas Spent => %d", gasSpent_);
    }
}

```

## Tools Used
Manual analysis

## Recommended Mitigation Steps


Increase the MIN_EXECUTION_OVERHEAD by:
- 20_000 for `RootBridgeAgent.anyExecute`.
- 110_000 for `anyExec` method in AnyCall.

20_000 + 110_000 = 130_000
So MIN_EXECUTION_OVERHEAD becomes 290_000 instead of 160_000. 

Additionally, calculate the gas consumption of the input data passed, add it to the cost.

Note: I suggest that the MIN_EXECUTION_OVERHEAD should be configurable/changeable. After launching OmniChain for some time, collect stats about the actual gas used for AnyCall on chain then adjust it accordingly. This also keeps you in the safe side in case any changes are applied on AnyCall contracts in future since it is upgradable.


## Assessed type

Other