## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-22

# [Multiple issues with `retrySettlement()` and `retrieveDeposit()` will cause loss of users' bridging deposits](https://github.com/code-423n4/2023-05-maia-findings/issues/356) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L441-L447
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L426
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1227-L1307


# Vulnerability details

Both `retrySettlement()` and `retrieveDeposit()` are incorrectly implemented with the following 3 issues, 

1. Both `retrySettlement()` and `retrieveDeposit()` are lacking a call to `wrappedNativeToken.deposit()` to wrap the native token paid by users for gas. This causes a subsequent call to `_depositGas()` to fail at [BranchBridgeAgent.sol#L929-L931](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L929-L931). This is also inconsistent with the other functions like `retryDeposit()`, which wraps the received native token for gas.
(see [BranchBridgeAgent.sol#L441-L447](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L441-L447))

2. `retrySettlement()` has a redundant increment for `depositNonce` in [BranchBridgeAgent.sol#L426](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L426), which will cause a different `depositNonce` value to be used for the subsequent call to `_createGasDeposit` in [BranchBridgeAgent.sol#L836](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L836).

3. Both `retrySettlement()` and `retrieveDeposit()` are missing fallback implementation, as `BranchBridgeAgent.anyFallback()` is not handling flag `0x07` (retrySettlement) and flag `0x08` (retrieveDeposit) as evident in [BranchBridgeAgent.sol#L1227-L1307](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L1227-L1307).


## Impact
Due to these issues, both `retrySettlement()` and `retrieveDeposit()` will cease to function properly. That will prevent users from re-trying failed settlement and retrieving deposits, resulting in loss of users' deposit for bridging. In addition, the gas paid by user that is not wrapped will also be stuck in BranchBridgeAgent as there is no function to withdraw native token.



## Proof of Concept

Add the following test case to `RootTest.t.sol`. This shows the issues with the lack of native token wrapping.

```Solidity
     function testPeakboltRetrySettlement() public {
        //Set up
        testAddLocalTokenArbitrum();

        //Prepare data
        bytes memory packedData;

        {
            Multicall2.Call[] memory calls = new Multicall2.Call[](1);

            //Mock action
            calls[0] = Multicall2.Call({target: 0x0000000000000000000000000000000000000000, callData: ""});

            //Output Params
            OutputParams memory outputParams = OutputParams(address(this), newAvaxAssetGlobalAddress, 150 ether, 0);

            //RLP Encode Calldata Call with no gas to bridge out and we top up.
            bytes memory data = abi.encode(calls, outputParams, ftmChainId);

            //Pack FuncId
            packedData = abi.encodePacked(bytes1(0x02), data);
        }

        address _user = address(this);

        //Get some gas.
        hevm.deal(_user, 1 ether);
        hevm.deal(address(ftmPort), 1 ether);

        //assure there is enough balance for mock action
        hevm.prank(address(rootPort));
        ERC20hTokenRoot(newAvaxAssetGlobalAddress).mint(address(rootPort), 50 ether, rootChainId);
        hevm.prank(address(avaxPort));
        ERC20hTokenBranch(avaxMockAssethToken).mint(_user, 50 ether);

        //Mint Underlying Token.
        avaxMockAssetToken.mint(_user, 100 ether);

        //Prepare deposit info
        DepositInput memory depositInput = DepositInput({
            hToken: address(avaxMockAssethToken),
            token: address(avaxMockAssetToken),
            amount: 150 ether,
            deposit: 100 ether,
            toChain: ftmChainId
        });


        console2.log("-------------  Creating a failed settlement ----------------");
        
        //Call Deposit function
        avaxMockAssetToken.approve(address(avaxPort), 100 ether);
        ERC20hTokenRoot(avaxMockAssethToken).approve(address(avaxPort), 50 ether);

        //Set MockAnycall AnyFallback mode ON
        MockAnycall(localAnyCallAddress).toggleFallback(1);

        //this is for branchBridgeAgent anyExecute
        uint128 remoteExecutionGas = 4e9;
        
        //msg.value is total gas amount for both Root and Branch agents
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: 1e11 }(packedData, depositInput, remoteExecutionGas);

        //Set MockAnycall AnyFallback mode OFF
        MockAnycall(localAnyCallAddress).toggleFallback(0);

        //Perform anyFallback transaction back to root bridge agent
        MockAnycall(localAnyCallAddress).testFallback();

        //check settlement status
        uint32 settlementNonce = multicallBridgeAgent.settlementNonce() - 1;
        Settlement memory settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);
        console2.log("Status after fallback:", settlement.status == SettlementStatus.Failed ? "Failed" : "Success");
        require(settlement.status == SettlementStatus.Failed, "Settlement status should be failed.");


        console2.log("------------- retrying Settlement ----------------");
        
        //Get some gas.
        hevm.deal(_user, 1 ether);

        //Retry Settlement        
        uint256 depositedGas = 7.9e9;
        uint128 gasToBridgeOut = 1.6e9;

        // This is expected to fail the gas paid by user is not wrapped and transfered 
        avaxMulticallBridgeAgent.retrySettlement{value: depositedGas}(settlementNonce, gasToBridgeOut);

        settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);

        require(settlement.status == SettlementStatus.Success, "Settlement status should be success.");

        address userAccount = address(RootPort(rootPort).getUserAccount(_user));
    


    }
```


## Recommended Mitigation Steps
1. Add `wrappedNativeToken.deposit{value: msg.value}();` to both `retrySettlement()` and `retrieveDeposit()`.
2. Remove the increment from `depositNonce` in [BranchBridgeAgent.sol#L426](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol#L426).
3. Add fallback implementation for both flag `0x07` (retrySettlement) and flag `0x08` (retrieveDeposit).




## Assessed type

Other