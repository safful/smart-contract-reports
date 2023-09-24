## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-31

# [Ulysses omnichain - RetrieveDeposit might never be able to Trigger the Fallback function](https://github.com/code-423n4/2023-05-maia-findings/issues/183) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/78e49c651fd119b85bb79296620d5cb39efa7cdd/src/ulysses-omnichain/RootBridgeAgent.sol#L1141-L1156


# Vulnerability details

## Impact
The purpose of the retrieveDeposit function is to enable a user to be able to redeem a deposit he entered into the system. the mechanism works based on the promise that this function will be able to forcefully make the root bridge agent trigger the fallback function. 

                if (!executionHistory[fromChainId][uint32(bytes4(data[1:5]))]) {
                //Toggle Nonce as executed
                executionHistory[fromChainId][nonce] = true;

                //Retry failed fallback
                (success, result) = (false, "")

by returning false, the anycall contract will attempt to trigger the fallback function in the branch bridge, which would in turn set the status of the deposit as failed. the user can then redeem his deposit because its status is now failed.

        function redeemDeposit(uint32 _depositNonce) external lock {
        //Update Deposit
        if (getDeposit[_depositNonce].status != DepositStatus.Failed) {
            revert DepositRedeemUnavailable();
        }

The problem is that according to how anycall protocol works, it is completely feasible that the execution in root bridge completes succesfully, but the fallback in branch might still fail to execute. 

     uint256 internal constant MIN_FALLBACK_RESERVE = 185_000; // 100_000 for anycall + 85_000 fallback execution overhead

for example, the anycall to the rootbridge might succeed due to enough gas stipend, while the fallback execution fails due to low gas stipend.

if this is the case then the deposit nonce would be stored in the executionHistory during the initial call, so when the retrievedeposit call is made it, it would think that the transaction is already completed, which would trigger this block instead:

                   _forceRevert();
                //Return true to avoid triggering anyFallback in case of `_forceRevert()` failure
                return (true, "already executed tx");


The impact of this is that if the deposit transaction is recorded in root side as completed, a user will never be able to use retrievedeposit function redeem his deposit from the system.


## Proof of Concept
    
    
     function testRetrieveDeposit() public {
        //Set up
        testAddLocalTokenArbitrum();

        //Prepare data
        bytes memory packedData;

        {
            Multicall2.Call[] memory calls = new Multicall2.Call[](1);

            //Mock action
            calls[0] = Multicall2.Call({target: 0x0000000000000000000000000000000000000000, callData:   ""});

            //Output Params
            OutputParams memory outputParams = OutputParams(address(this), newAvaxAssetGlobalAddress, 150 ether, 0);

            //RLP Encode Calldata Call with no gas to bridge out and we top up.
            bytes memory data = abi.encode(calls, outputParams, ftmChainId);

            //Pack FuncId
            packedData = abi.encodePacked(bytes1(0x02), data);
        }

        address _user = address(this);

        //Get some gas.
        hevm.deal(_user, 100 ether);
        hevm.deal(address(ftmPort), 1 ether);

        //assure there is enough balance for mock action
        hevm.prank(address(rootPort));
        ERC20hTokenRoot(newAvaxAssetGlobalAddress).mint(address(rootPort), 50 ether, rootChainId);
        hevm.prank(address(avaxPort));
        ERC20hTokenBranch(avaxMockAssethToken).mint(_user, 50 ether);

        //Mint Underlying Token.
        avaxMockAssetToken.mint(_user, 100 ether);

        //Prepare deposit info
                //Prepare deposit info
        DepositParams memory depositParams = DepositParams({
            hToken: address(avaxMockAssethToken),
            token: address(avaxMockAssetToken),
            amount: 150 ether,
            deposit: 100 ether,
            toChain: ftmChainId,
            depositNonce: 1,
            depositedGas: 1 ether
        });

        DepositInput memory depositInput = DepositInput({
            hToken: address(avaxMockAssethToken),
            token: address(avaxMockAssetToken),
            amount: 150 ether,
            deposit: 100 ether,
            toChain: ftmChainId
        });

        // Encode AnyFallback message
        bytes memory anyFallbackData = abi.encodePacked(
            bytes1(0x01),
            depositParams.depositNonce,
            depositParams.hToken,
            depositParams.token,
            depositParams.amount,
            depositParams.deposit,
            depositParams.toChain,
            bytes("testdata"),
            depositParams.depositedGas,
            depositParams.depositedGas / 2
        );

        console2.log("BALANCE BEFORE:");
        console2.log("User avaxMockAssetToken Balance:", MockERC20(avaxMockAssetToken).balanceOf(_user));
        console2.log("User avaxMockAssethToken Balance:",  MockERC20(avaxMockAssethToken).balanceOf(_user));

        require(avaxMockAssetToken.balanceOf(address(avaxPort)) == 0, "balance of port is not zero");

        //Call Deposit function
        avaxMockAssetToken.approve(address(avaxPort), 100 ether);
        ERC20hTokenRoot(avaxMockAssethToken).approve(address(avaxPort), 50 ether);
        avaxMulticallBridgeAgent.callOutSignedAndBridge{value: 50 ether}(packedData, depositInput, 0.5 ether);
    ;
        

        avaxMulticallBridgeAgent.retrieveDeposit{value: 1 ether}(depositParams.depositNonce);


        // fallback is not triggered.

        // @audit Redeem Deposit, will fail with   DepositRedeemUnavailable()
        avaxMulticallBridgeAgent.redeemDeposit(depositParams.depositNonce);

    }

## Tools Used
Manual Review

## Recommended Mitigation Steps

just make the root bridge return (false, "") regardles of whether the transaction linked to the original deposit was completed or not. 

     /// DEPOSIT FLAG: 8 (retrieveDeposit)
    else if (flag == 0x08) {
            (success, result) = (false, "");

to avoid also spamming the usage of the retrievedeposit function, it is advisable to add a check in the retrieveDeposit function to see whether the deposit still exists, it doesnt make sense to try and retrieve a deposit that has already been redeemed.

        function retrieveDeposit(uint32 _depositNonce) external payable lock requiresFallbackGas {
        address depositOwner = getDeposit[_depositNonce].owner;
        
        if (depositOwner == address(0)) {
            revert RetrieveDepositUnavailable();
        }














## Assessed type

Other