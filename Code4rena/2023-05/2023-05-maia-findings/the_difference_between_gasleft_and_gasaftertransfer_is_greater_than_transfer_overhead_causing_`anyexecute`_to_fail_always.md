## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-15

# [The difference between gasLeft and gasAfterTransfer is greater than TRANSFER_OVERHEAD causing `anyExecute` to fail always](https://github.com/code-423n4/2023-05-maia-findings/issues/610) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1029-L1054


# Vulnerability details

## Impact
In `_payExecutionGas` ,  there is the following code:

```solidity
///Save gas left
	uint256 gasLeft = gasleft();
	.
	.
	. 
	.
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
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1029-L1054

It checks if the difference between gasLeft and gasAfterTransfer is greater than TRANSFER_OVERHEAD then it calls _forceRevert(). So that Anycall Executor reverts the call. This check has been Introduced to prevent any arbitrary code executed in the  _recipient's fallback (This was confirmed by the sponsor). However, the condition `gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD` is always true. TRANSFER_OVERHEAD is 24_000
```solidity
uint256 internal constant TRANSFER_OVERHEAD = 24_000;
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L139

And the **gas spent between gasLeft and gasAfterTransfer is nearly 70_000 which is higher than 24_000**. Thus, causing the function to revert always.  `_payExecutionGas` is called by `anyExecute` which is called by Anycall Executor. This means `anyExecute` will also fail. This happens because gasLeft value is stored before even replenishing gas and not before transfer.

## Proof of Concept

#### Overview
This PoC is independent from the codebase (but uses the same code).
There are one contract simulating `BranchBridgeAgent.anyExecute`.

We run the test, then anyExecute will revert because gasLeft - gasAfterTransfer is always greater than TRANSFER_OVERHEAD (24_000).

Here is the output of the test:

```sh
[PASS] test_anyexecute_always_revert_bc_transfer_overhead() (gas: 124174)
Logs:
  (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD) => true
  gasLeft - gasAfterTransfer = 999999999999979606 - 999999999999909238 = 70368

Test result: ok. 1 passed; 0 failed; finished in 1.88ms
```

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
- In `_forceRevert`, we call anycallConfig Immediately skippping the returned value from AnycallProxy. This is anyway irrelevant for this PoC.

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
// PoC => Maia OmniChain: anyExecute always revert in BranchBridgeAgent
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
    function forceSafeTransferETH(
        address to,
        uint256 amount,
        uint256 gasStipend
    ) internal {
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
                    if iszero(gt(gas(), 1000000)) {
                        revert(0, 0)
                    }
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
                    if iszero(gt(gas(), 1000000)) {
                        revert(0, 0)
                    }
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
    function trySafeTransferETH(
        address to,
        uint256 amount,
        uint256 gasStipend
    ) internal returns (bool success) {
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
    function safeTransferFrom(
        address token,
        address from,
        address to,
        uint256 amount
    ) internal {
        /// @solidity memory-safe-assembly
        assembly {
            let m := mload(0x40) // Cache the free memory pointer.

            mstore(0x60, amount) // Store the `amount` argument.
            mstore(0x40, to) // Store the `to` argument.
            mstore(0x2c, shl(96, from)) // Store the `from` argument.
            // Store the function selector of `transferFrom(address,address,uint256)`.
            mstore(0x0c, 0x23b872dd000000000000000000000000)

            if iszero(
                and(
                    // The arguments of `and` are evaluated from right to left.
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
    function safeTransferAllFrom(
        address token,
        address from,
        address to
    ) internal returns (uint256 amount) {
        /// @solidity memory-safe-assembly
        assembly {
            let m := mload(0x40) // Cache the free memory pointer.

            mstore(0x40, to) // Store the `to` argument.
            mstore(0x2c, shl(96, from)) // Store the `from` argument.
            // Store the function selector of `balanceOf(address)`.
            mstore(0x0c, 0x70a08231000000000000000000000000)
            if iszero(
                and(
                    // The arguments of `and` are evaluated from right to left.
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
                and(
                    // The arguments of `and` are evaluated from right to left.
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
                and(
                    // The arguments of `and` are evaluated from right to left.
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
    function safeTransferAll(
        address token,
        address to
    ) internal returns (uint256 amount) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x00, 0x70a08231) // Store the function selector of `balanceOf(address)`.
            mstore(0x20, address()) // Store the address of the current contract.
            if iszero(
                and(
                    // The arguments of `and` are evaluated from right to left.
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
                and(
                    // The arguments of `and` are evaluated from right to left.
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
                and(
                    // The arguments of `and` are evaluated from right to left.
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
    function balanceOf(
        address token,
        address account
    ) internal view returns (uint256 amount) {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x14, account) // Store the `account` argument.
            // Store the function selector of `balanceOf(address)`.
            mstore(0x00, 0x70a08231000000000000000000000000)
            amount := mul(
                mload(0x20),
                and(
                    // The arguments of `and` are evaluated from right to left.
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
        if (msg.sender != localAnyCallExecutorAddress)
            revert AnycallUnauthorizedCaller();
        (address from, , ) = IAnycallExecutor(localAnyCallExecutorAddress)
            .context();
        if (from != rootBridgeAgentAddress) revert AnycallUnauthorizedCaller();
    }

    function _replenishGas(uint256 _executionGasSpent) internal virtual {
        //Deposit Gas
        anycallV7Config.deposit{value: _executionGasSpent}(address(this));
        // IAnycallConfig(IAnycallProxy(localAnyCallAddress).config()).deposit{value: _executionGasSpent}(address(this));
    }

    function _forceRevert() internal virtual {
        IAnycallConfig anycallConfig = IAnycallConfig(
            IAnycallProxy(localAnyCallAddress).config()
        );

        // uint256 executionBudget = anycallConfig.executionBudget(address(this));
        uint256 executionBudget = anycallV7Config.executionBudget(
            address(this)
        );

        // Withdraw all execution gas budget from anycall for tx to revert with "no enough budget"
        if (executionBudget > 0)
            try anycallConfig.withdraw(executionBudget) {} catch {}
    }

    /**
     * @notice Internal function repays gas used by Branch Bridge Agent to fulfill remote initiated interaction.
     *   @param _recipient address to send excess gas to.
     *   @param _initialGas gas used by Branch Bridge Agent.
     */
    function _payExecutionGas(
        address _recipient,
        uint256 _initialGas
    ) internal virtual {
        //Gas remaining
        uint256 gasRemaining = wrappedNativeToken.balanceOf(address(this));

        //Unwrap Gas
        wrappedNativeToken.withdraw(gasRemaining);

        //Delete Remote Initiated Action State
        delete (remoteCallDepositedGas);

        ///Save gas left
        uint256 gasLeft = gasleft();

        //Get Branch Environment Execution Cost
        // Assume tx.gasPrice 1e9
        uint256 minExecCost = 1e9 *
            (MIN_EXECUTION_OVERHEAD + _initialGas - gasLeft);

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

        //Check if sufficient balance // This condition is always true
        if (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD) {
            console.log(
                "(gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD) => true"
            );
            console.log(
                "gasLeft - gasAfterTransfer = %d - %d = %d",
                gasLeft,
                gasAfterTransfer,
                gasLeft - gasAfterTransfer
            );
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
        address recipient = address(0x0); // for simplicity and since it is irrelevant //address(uint160(bytes20(data[PARAMS_START:PARAMS_START_SIGNED])));

        // Other Code Here

        //Deduct gas costs from deposit and replenish this bridge agent's execution budget.
        _payExecutionGas(recipient, initialGas);
    }

    function depositIntoWeth(uint256 amt) external {
        wrappedNativeToken.deposit{value: amt}();
    }

    fallback() external payable {}
}

contract GasCalcTransferOverHead is DSTest, Test {
    BranchBridgeAgent branchBridgeAgent;

    function setUp() public {
        branchBridgeAgent = new BranchBridgeAgent();
        vm.deal(
            address(branchBridgeAgent.localAnyCallExecutorAddress()),
            100 ether
        ); // executer pays gas
        vm.deal(address(branchBridgeAgent), 100 ether);
    }

    function test_anyexecute_always_revert_bc_transfer_overhead() public {
        // add weth balance
        branchBridgeAgent.depositIntoWeth(100 ether);
        vm.prank(address(branchBridgeAgent.localAnyCallExecutorAddress()));
        vm.expectRevert();
        branchBridgeAgent.anyExecute{gas: 1 ether}(bytes(""));
        vm.stopPrank();
    }
}

```

## Tools Used
Manual analysis

## Recommended Mitigation Steps

Increase the TRANSFER_OVERHEAD to cover the actual gas spent.
You could also add a gas check point immediately before the transfer to make the naming makes sense (i.e. TRANSFER_OVERHEAD). However, the gas will be nearly 34_378 which is still be higher than TRANSFER_OVERHEAD (24_000).

You can simply comment out the code after gasLeft till the transfer, remove `- minExecCost` from the value to transfer since it is commented out.
Now run the test again, you will see an output like this (with failed test but we are not interested in it anyway):
```solidity
[FAIL. Reason: Call did not revert as expected] test_anyexecute_always_revert_bc_transfer_overhead() (gas: 111185)
Logs:
  (gasLeft - gasAfterTransfer > TRANSFER_OVERHEAD) => true
  gasLeft - gasAfterTransfer = 999999999999979606 - 999999999999945228 = 34378

Test result: FAILED. 0 passed; 1 failed; finished in 1.26ms
```

**gasLeft - gasAfterTransfer = 34378**

Please note that I have tested a simple function in Remix as well and it gave the same gas spent (i.e. 34378)

```
// copy the library code from Solady and paste it here
// https://github.com/Vectorized/solady/blob/main/src/utils/SafeTransferLib.sol


contract Test {

       function testGas() payable public returns (uint256){
        ///Save gas left
        uint256 gasLeft = gasleft();

        //Transfer gas remaining to recipient
        SafeTransferLib.safeTransferETH(address(0), 1 ether);

        //Save Gas
        uint256 gasAfterTransfer = gasleft();

        return gasLeft-gasAfterTransfer;

       }

}
```


The returned value will be 34378


## Assessed type

Other