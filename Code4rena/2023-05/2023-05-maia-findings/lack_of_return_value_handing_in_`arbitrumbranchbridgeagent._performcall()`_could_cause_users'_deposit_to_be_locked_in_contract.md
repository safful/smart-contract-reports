## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-27

# [Lack of return value handing in `ArbitrumBranchBridgeAgent._performCall()` could cause users' deposit to be locked in contract](https://github.com/code-423n4/2023-05-maia-findings/issues/266) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchBridgeAgent.sol#L143


# Vulnerability details

In `ArbitrumBranchBridgeAgent`, the `_performCall()` is overridden to directly call `RootBridgeAgent.anyExecute()` instead of performing an AnyCall cross-chain transaction as `RootBridgeAgent` is also in Arbitrum. However, unlike AnyCall, `ArbitrumBranchBridgeAgent._performCall()` is missing the handling of return value for `anyExecute()`.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/ArbitrumBranchBridgeAgent.sol#L143
```Solidity
 function _performCall(bytes memory _callData) internal override {
        IRootBridgeAgent(rootBridgeAgentAddress).anyExecute(_callData);
    }
```

That is undesirable as `RootBridgeAgent.anyExecute()` has a try/catch that prevents revert from bubbling up. Instead, it expects `ArbitrumBranchBridgeAgent._performCall()` to revert when `success == false`, which is currently missing.

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L1068-L1074
```Solidity
            try RootBridgeAgentExecutor(bridgeAgentExecutorAddress).executeSignedWithDeposit(
                address(userAccount), localRouterAddress, data, fromChainId
            ) returns (bool, bytes memory res) {
                (success, result) = (true, res);
            } catch (bytes memory reason) {
                result = reason;
            }
```

## Impact


Without handling the scenario when `RootBridgeAgent.anyExecute()` returns false, `ArbitrumBranchBridgeAgent._performCall()` will continue execution even for failed calls and not revert due to the try/catch in `RootBridgeAgent.anyExecute()`.

In the worst case, users could lose their bridged deposit when they use `ArbitrumBranchBridgeAgent.callOutSignedAndBridge()` to interact with dApps and encountered failed calls. 

When failed calls to dApps occur, `ArbitrumBranchBridgeAgent.callOutSignedAndBridge()` is expected to revert the entire transaction and reverse the bridging of deposit. However, due to the issue with `_performCall()`, the bridged deposit will not be reverted, thus locking up users' fund in the contract. Furthermore, `RootBridgeAgent.anyExecute()` will mark the deposit transaction as executed in `executionHistory[]`, preventing any `retryDeposit()` or `retrieveDeposit()` attempts to recover the funds.
## Proof of Concept

Add the following MockContract and test case to ArbitrumBranchTest.t.sol and run the test case.

```Solidity
    contract MockContract is Test {
        function test() external {
            require(false);
        }
    } 

    function testPeakboltArbCallOutWithDeposit() public {
        //Set up
        testAddLocalTokenArbitrum();

        // deploy mock contract to call using multicall
        MockContract mockContract = new MockContract();

        //Prepare data
        address outputToken;
        uint256 amountOut;
        uint256 depositOut;
        bytes memory packedData;

        {
            outputToken = newArbitrumAssetGlobalAddress;
            amountOut = 100 ether;
            depositOut = 50 ether;

            Multicall2.Call[] memory calls = new Multicall2.Call[](1);

            //prepare for a call to MockContract.test(), which will revert
            calls[0] = Multicall2.Call({target: address(mockContract), callData: abi.encodeWithSignature("test()")});
    
            //Output Params
            OutputParams memory outputParams = OutputParams(address(this), outputToken, amountOut, depositOut);

            //toChain
            uint24 toChain = rootChainId;

            //RLP Encode Calldata
            bytes memory data = abi.encode(calls, outputParams, toChain);

            //Pack FuncId
            packedData = abi.encodePacked(bytes1(0x02), data);
        }

        //Get some gas.
        hevm.deal(address(this), 1 ether);

        //Mint Underlying Token.
        arbitrumNativeToken.mint(address(this), 100 ether);

        //Approve spend by router
        arbitrumNativeToken.approve(address(localPortAddress), 100 ether);

        //Prepare deposit info
        DepositInput memory depositInput = DepositInput({
            hToken: address(newArbitrumAssetGlobalAddress),
            token: address(arbitrumNativeToken),
            amount: 100 ether,
            deposit: 100 ether,
            toChain: rootChainId
        });

        //Mock messaging layer fees
        hevm.mockCall(
            address(localAnyCongfig),
            abi.encodeWithSignature("calcSrcFees(address,uint256,uint256)", address(0), 0, 100),
            abi.encode(0)
        );

        console2.log("Initial User Balance: %d", arbitrumNativeToken.balanceOf(address(this)));

        //Call Deposit function
        arbitrumMulticallBridgeAgent.callOutSignedAndBridge{value: 1 ether}(packedData, depositInput, 0.5 ether);

        // This shows that deposit entry is successfully created
        testCreateDepositSingle(
            arbitrumMulticallBridgeAgent,
            uint32(1),
            address(this),
            address(newArbitrumAssetGlobalAddress),
            address(arbitrumNativeToken),
            100 ether,
            100 ether,
            1 ether,
            0.5 ether
        );

        // The following shows that user deposited to the LocalPort, but it is not deposited/bridged to the user account
        console2.log("LocalPort Balance (expected):", uint256(50 ether));
        console2.log("LocalPort Balance (actual):", MockERC20(arbitrumNativeToken).balanceOf(address(localPortAddress)));
        //require(MockERC20(arbitrumNativeToken).balanceOf(address(localPortAddress)) == 50 ether, "LocalPort should have 50 tokens");

        console2.log("User Balance: (expected)", uint256(50 ether));
        console2.log("User Balance: (actual)", MockERC20(arbitrumNativeToken).balanceOf(address(this)));
        //require(MockERC20(arbitrumNativeToken).balanceOf(address(this)) == 50 ether, "User should have 50 tokens");

        console2.log("User Global Balance: (expected)", uint256(50 ether));
        console2.log("User Global Balance: (actual)", MockERC20(newArbitrumAssetGlobalAddress).balanceOf(address(this)));
        //require(MockERC20(newArbitrumAssetGlobalAddress).balanceOf(address(this)) == 50 ether, "User should have 50 global tokens");

        // retryDeposit() will fail as well as the the transaction is marked executed in executionHistory
        uint32 depositNonce = arbitrumMulticallBridgeAgent.depositNonce() - 1;
        hevm.deal(address(this), 1 ether);
        //hevm.expectRevert(abi.encodeWithSignature("GasErrorOrRepeatedTx()"));
        arbitrumMulticallBridgeAgent.retryDeposit{value: 1 ether}(true, depositNonce, "", 0.5 ether, rootChainId);

    }
```


## Recommended Mitigation Steps
Handle the return value of `anyExecute()` in `_performCall()` and revert on `success == false`.


## Assessed type

Other