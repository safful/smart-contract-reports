## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-21

# [Missing unwrapping of native token in RootBridgeAgent.sweep() causes fees to be stuck](https://github.com/code-423n4/2023-05-maia-findings/issues/385) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L1259-L1264


# Vulnerability details


`RootBridgeAgent.sweep()` will fail as it tries to transfer `accumulatedFees` using `SafeTransferLib.safeTransferETH()` but fails to unwrap the fees by withdrawing from `wrappedNativeToken`.

## Impact
The `accumulatedFees` will be stuck in `RootBridgeAgent` without any functions to withdraw them.

## Proof of Concept
Add the below test case to `RootTest.t.sol`.

    function testPeakboltSweep() public {
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

        console2.log("accumulatedFees (BEFORE) = %d", multicallBridgeAgent.accumulatedFees());       

        //Call Deposit function
        avaxMockAssetToken.approve(address(avaxPort), 100 ether);
        ERC20hTokenRoot(avaxMockAssethToken).approve(address(avaxPort), 50 ether);

        uint128 remoteExecutionGas = 4e9;
        uint128 depositedGas = 1e11; 
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: depositedGas }(packedData, depositInput, remoteExecutionGas);

        console2.log("accumulatedFees (AFTER)  = %d", multicallBridgeAgent.accumulatedFees());        
        console2.log("WETH Balance = %d ", multicallBridgeAgent.wrappedNativeToken().balanceOf(address(multicallBridgeAgent)));
        console2.log("ETH Balance = %d ", address(multicallBridgeAgent).balance);

        // sweep() will fail as it does not unwrap the WETH before the ETH transfer
        multicallBridgeAgent.sweep();

    }

## Recommended Mitigation Steps
Add `wrappedNativeToken.withdraw(_accumulatedFees);` to `sweep()` before transfering.



## Assessed type

Other