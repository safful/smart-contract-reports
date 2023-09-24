## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-27

# [Ulysses omnichain - User Funds can get locked permanently via making callout without deposit](https://github.com/code-423n4/2023-05-maia-findings/issues/377) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/78e49c651fd119b85bb79296620d5cb39efa7cdd/src/ulysses-omnichain/MulticallRootRouter.sol#L175-L236


# Vulnerability details

## Impact
ulyssses omnichain provides the user with the ability to do what is known as multicall transactions. this is possible via the multicallrouter which from my review of code base, is the protocols primarily method for enabling omnichain transactions between source and destination chains. the contract enables the user who is in the source chain, lets say avax, to do multicalls in the root chain via their virtual account, withdraw funds from their virtual account and then use these funds to do multiple settlements(or a single settlement) in destination chain which could be FTM for instance,and lets them retrieve their desired output tokens based on amounts they deposited, all of this within a single transaction.

There are multiple endpoints exposed to enable these multicall functionalities, one of them enables what is known as a multicallmultioutputnodeposit or multicallsingleoutputnodeposit. they are exposed to branch bridge agents via calling 0x01 flag to signal a transaction without a deposit. this in turn reaches the root, and triggers the executeNoDeposit function in the rootbranchbridgeexecutor contract. that function then fires the anyExecute function in the multicallrouter contract. 

        function anyExecute(bytes1 funcId, bytes calldata encodedData, uint24)
        external
        payable
        override
        lock
        requiresExecutor
        returns (bool, bytes memory)
    {

based on the payload set by user from the source chain, the function will do a number of things, first it determines the type of transaction via the flag, lets assume the user chose the multicallMultipleOutput, which signals he wants to do multiple calls within root environment, and then finally he wants to do multiple settlements in destination chain. this would trigger the code block below:

                /// FUNC ID: 2 (multicallSingleOutput)
        } else if (funcId == 0x02) {
            (IMulticall.Call[] memory callData, OutputParams memory outputParams, uint24 toChain) =
                abi.decode(encodedData, (IMulticall.Call[], OutputParams, uint24));

            _multicall(callData);

            _approveAndCallOut(
                address(0), // @audit should this be address of user who initiates request?, so that he can retry a settlement that failed to fire fallback
                outputParams.recipient,
                outputParams.outputToken,
                outputParams.amountOut,
                outputParams.depositOut,
                toChain
            );

            /// FUNC ID: 3 (multicallMultipleOutput)
        } else if (funcId == 0x03) {
            (IMulticall.Call[] memory callData, OutputMultipleParams memory outputParams, uint24 toChain) =
                abi.decode(encodedData, (IMulticall.Call[], OutputMultipleParams, uint24));

            _multicall(callData);

            _approveMultipleAndCallOut(
                address(0), // @audit should this be address of user who initiates request?, so that he can retry a settlement that failed to fire fallback
                outputParams.recipient,
                outputParams.outputTokens,
                outputParams.amountsOut,
                outputParams.depositsOut,
                toChain
            );
            /// UNRECOGNIZED FUNC ID
        }


the problem manifests in the code block above, essentially because the owner of the settlement that needs to be cleared in destination branch will be the zero address. depending on whether user requested a multicallSingleOutput or a multicallMultiOutput action, this code block will do a number of things, first it will allow the user to make multi calls via payload he specified, second it will approve the Root Port to spend output hTokens on behalf of user. it will then move output hTokens from Root to destination Branch and call 'clearTokens'. this process updates the state of the root bridge via the _updateStateOnBridgeOut. the tokens are then 'cleared' in destination chain, and user should receive his desired output tokens. 

However, because the settlement has no linked owner, if the transaction to destination chain fails for whatever reason, the user will be unable to retry the settlement via the root bridge, and they also cannot redeem the settlement. this effectively means the user funds transfered to ulysses root environment  are essentialy locked.

        function redeemSettlement(uint32 _depositNonce) external lock {
        //Get deposit owner.
        address depositOwner = getSettlement[_depositNonce].owner;

        //Update Deposit
        if (getSettlement[_depositNonce].status != SettlementStatus.Failed || depositOwner == address(0)) {
            revert SettlementRedeemUnavailable();

as you can see above, a settlement with owner set to address zero is not redeemable. the logic behind that was a redeemed settlement will be deleted and hence  getSettlement would retrieve an empty settlement struct with an owner of address zero.

    
    function _retrySettlement(uint32 _settlementNonce) internal returns (bool) {
        //Get Settlement
        Settlement memory settlement = getSettlement[_settlementNonce];

        //Check if Settlement hasn't been redeemed.
        if (settlement.owner == address(0)) return false;

as you can see above, settlement retries will also fail, because once again, the settlement was set with owner of address zero initially.

the impact of this is very high in my opinion because not only will user funds be permanently locked, but system invariants will be broken since the token accounting in system will not be in balance.  proof of concept will help clarify this issue further.
 

## Proof of Concept

        function testMulticallMultipleOutputNoDepositFailed() public {
        //Add Local Token from Avax
        testSetLocalToken();

        require(
            RootPort(rootPort).getLocalTokenFromGlobal(newAvaxAssetGlobalAddress, ftmChainId) == newAvaxAssetLocalToken,
            "Token should be added"
        );

        
        hevm.deal(address(userVirtualAccount), 1 ether);
        hevm.deal(address(avaxMulticallBridgeAgentAddress), 10 ether);

        //Prepare data
        address[] memory outputTokens = new address[](2);
        uint256[] memory amountsOut = new uint256[](2);
        uint256[] memory depositsOut = new uint256[](2);
        bytes memory packedData;

        {
            outputTokens[0] = ftmGlobalToken;
            outputTokens[1] = newAvaxAssetGlobalAddress;
            amountsOut[0] = 100 ether;
            amountsOut[1] = 100 ether;
            depositsOut[0] = 50 ether;
            depositsOut[1] = 0 ether;

            Multicall2.Call[] memory calls = new Multicall2.Call[](2);

            //Prepare call to transfer 100 wFTM global token from contract to Root Multicall Router
            calls[0] = Multicall2.Call({
                target: ftmGlobalToken,
                callData: abi.encodeWithSelector(bytes4(0x23b872dd), address(userVirtualAccount), address(rootMulticallRouter), 100 ether)
            });

            //Prepare call to transfer 100 hAVAX global token from contract to Root Multicall Router
            calls[1] = Multicall2.Call({
                target: newAvaxAssetGlobalAddress,
                callData: abi.encodeWithSelector(bytes4(0x23b872dd), address(userVirtualAccount), address(rootMulticallRouter), 100 ether)
            });


            //Output Params
            OutputMultipleParams memory outputMultipleParams =
                OutputMultipleParams(userVirtualAccount, outputTokens, amountsOut, depositsOut);

            // minted assets to the user directly
            hevm.startPrank(address(rootPort));
            ERC20hTokenRoot(ftmGlobalToken).mint(userVirtualAccount, 100 ether, ftmChainId);
            ERC20hTokenRoot(newAvaxAssetGlobalAddress).mint(userVirtualAccount, 100 ether, avaxChainId);
            hevm.stopPrank();


            uint256 balanceUserBeforeAvax = MockERC20(newAvaxAssetGlobalAddress).balanceOf(userVirtualAccount);
            uint256 balanceUserBeforeFtm =  MockERC20(ftmGlobalToken).balanceOf(userVirtualAccount);
            require(balanceUserBeforeAvax == 100 ether, "User Balance should be 100 avax");
            require(balanceUserBeforeFtm == 100 ether, "User Balance should be 100 ftm");

            //User Approves spend by multicall contract
            hevm.startPrank(address(userVirtualAccount));
            MockERC20(ftmGlobalToken).approve(address(rootMulticallRouter.multicallAddress()), 100 ether);
            MockERC20(newAvaxAssetGlobalAddress).approve(address(rootMulticallRouter.multicallAddress()), 100 ether);
            hevm.stopPrank();

            //toChain
            uint24 toChain = ftmChainId;

            //RLP Encode Calldata
            bytes memory data = abi.encode(calls, outputMultipleParams, toChain);

            //Pack FuncId
            packedData = abi.encodePacked(bytes1(0x03), data);
            
        }



        uint256 balanceBeforePortAvax = MockERC20(newAvaxAssetGlobalAddress).balanceOf(address(rootPort));
        uint256 balanceBeforePortFtm = MockERC20(ftmGlobalToken).balanceOf(address(rootPort));


        
        //Call Deposit function
        encodeCallNoDeposit(
            payable(avaxMulticallBridgeAgentAddress),
            payable(multicallBridgeAgent),
            1,
            packedData,
            0.0001 ether,
            0.00005 ether,
            avaxChainId
        );


        uint256 balanceUserAfterAvax = MockERC20(newAvaxAssetGlobalAddress).balanceOf(userVirtualAccount);
        uint256 balanceUserAfterFtm = MockERC20(ftmGlobalToken).balanceOf(userVirtualAccount);
        require(balanceUserAfterAvax == 0 ether, "User Balance should be 0 global avax");
        require(balanceUserAfterFtm == 0 ether, "User Balance should be 0 global ftm");


        uint256 balanceAfter = MockERC20(newAvaxAssetGlobalAddress).balanceOf(address(rootMulticallRouter));
        uint256 balanceFtmAfter = MockERC20(ftmGlobalToken).balanceOf(address(rootMulticallRouter));

        require(balanceAfter == 0, "Router Balance should be cleared");
        require(balanceFtmAfter == 0, "Router Balance should be cleared");


        uint256 balanceAfterPortAvax = MockERC20(newAvaxAssetGlobalAddress).balanceOf(address(rootPort));
        uint256 balanceAfterPortFtm = MockERC20(ftmGlobalToken).balanceOf(address(rootPort));

        require(balanceAfterPortAvax ==  balanceBeforePortAvax + 100 ether, "Root port global avax Balance should be increased by 100");
        require(balanceAfterPortFtm ==  balanceBeforePortFtm + 50 ether, "Root port global ftm Balance should be increased by 50");

        uint32 settlementNonce = 1;

        Settlement memory settlement = multicallBridgeAgent.getSettlementEntry(settlementNonce);

        console2.log("Status after fallback:", settlement.status == SettlementStatus.Failed ? "Failed" : "Success");

        require(settlement.status == SettlementStatus.Success, "Settlement status should be success.");

       
        bytes memory anyFallbackData = abi.encodePacked(bytes1(0x01), address(userVirtualAccount), uint32(settlementNonce), packedData, uint128(0.0001 ether), uint128(0.00005 ether));
        hevm.prank(localAnyCallExecutorAddress);
        multicallBridgeAgent.anyFallback(anyFallbackData);

        Settlement memory settlement3 = multicallBridgeAgent.getSettlementEntry(settlementNonce);

        console2.log("Status after fallback:", settlement3.status == SettlementStatus.Failed ? "Failed" : "Success");

        require(settlement3.status == SettlementStatus.Failed, "Settlement status should be failed after fallback.");

        //Attempt to Redeem settlement since its now in failed state via fallback
        hevm.startPrank(address(userVirtualAccount));
        // @audit this will fail with SettlementRedeemUnavailable() since settlement has no owner
        multicallBridgeAgent.redeemSettlement(settlementNonce);
        hevm.stopPrank();


    }

The POC above demonstrates in detail how this problem develops. in this single transaction it is possible for a user to leverage the multicall feature to transfer their own funds to the rootMulticallRouter, which would then proceed to attempt to settle transactions in the destination chain the user chose.  
 
The following discussion is based on POC inputs: 

For FTM , user is requesting an amount of 100 and deposit of 50, this will cause the root bridge agent to update its state, which will effectively increase ts port balance of FTM global by 50, it will also effectively burn the remaining 50 that is in the multi router.

For Avax Global , user is requesting an amount of 100 and deposit of 0, this will cause the root bridge agent to update its state, which will effectively increase its port balance of AVAX global by 100, the full amount.


If the transfer to FTM bridge agent is successful, it will settle the transaction for the user in the destination chain, which will do the following:

For FTM global, it will bridge in(mint) 50 local FTM tokens for the user. it will also withdraw 50 global tokens and give them to the user. so the user ends up with the same amount of tokens he started with, except he now has 50 global FTM and 50 local FTM.


For AVAX global, it will bridge in or mint 100 local AVAX tokens for the user. so the user ends up with the same amount of tokens he started with, except he now has 100 local AVAX tokens.

But what if the calloutandbridge from root to branch bridge agent fails, maybe to low gas. The anyfallback if fired successfully in root bridge, will enable a user to either retry or redeem the settlement. The problem is the _approveAndCallOut function the multicallrootrouter set the owner to the zero address. which means the owner of the settlement will be the zero address. This means the user will have no way to retry or redeem his settlement in the destination branch branch. This effectively means the user has lost the entire amount of global AVAX and FTM he initially deposited with the router to process his request. Not only that but the token accounting in the system will not be in balance.for example the total amount of global AVAX in the system will not equal the total amount of local AVAX, hence the invariant of 1:1 supply is broken. The broken invariant applies to FTM as well. specifically ftm global > ftm local by 50, and AVAX global > AVAX local by 100.


## Tools Used
Manual Review

## Recommended Mitigation Steps

                _approveAndCallOut(
                outputParams.recipient, // @audit: fixed here by updating the owner of settlement
                outputParams.recipient,
                outputParams.outputToken,
                outputParams.amountOut,
                outputParams.depositOut,
                toChain
            );

run the poc again with this modification, it should pass. 




















## Assessed type

Other