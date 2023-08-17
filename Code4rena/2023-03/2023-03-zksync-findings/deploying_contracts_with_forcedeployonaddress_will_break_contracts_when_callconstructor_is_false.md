## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- M-01

# [deploying contracts with forceDeployOnAddress will break contracts when callConstructor is false](https://github.com/code-423n4/2023-03-zksync-findings/issues/167) 

# Lines of code

https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L212-L227
https://github.com/code-423n4/2023-03-zksync/blob/21d9a364a4a75adfa6f1e038232d8c0f39858a64/contracts/ContractDeployer.sol#L302-L306


# Vulnerability details

## Impact
when function `forceDeployOnAddress()` used for deploying contract and `callConstructor` is false, then contract's bytecodehash would stay in constructing state and calling the contract won't be possible. it can cause protocol and other contracts that are using it to break and if they call that address and sends some funds, then those funds would be lost. the issue is critical because the updated contract(which is updated by calling `forceDeployOnAddress()`) can be part of important process like bridging or sending messages or withdrawing funds.

## Proof of Concept
This is `forceDeployOnAddress()` code in ContractDeployer:
```solidity
    /// @notice The method that can be used to forcefully deploy a contract.
    /// @param _deployment Information about the forced deployment.
    /// @param _sender The `msg.sender` inside the constructor call.
    function forceDeployOnAddress(ForceDeployment calldata _deployment, address _sender) external payable onlySelf {
        _ensureBytecodeIsKnown(_deployment.bytecodeHash);
        _storeConstructingByteCodeHashOnAddress(_deployment.newAddress, _deployment.bytecodeHash);

        AccountInfo memory newAccountInfo;
        newAccountInfo.supportedAAVersion = AccountAbstractionVersion.None;
        // Accounts have sequential nonces by default.
        newAccountInfo.nonceOrdering = AccountNonceOrdering.Sequential;
        _storeAccountInfo(_deployment.newAddress, newAccountInfo);

        if (_deployment.callConstructor) {
            _constructContract(_sender, _deployment.newAddress, _deployment.input, false);
        }

        emit ContractDeployed(_sender, _deployment.bytecodeHash, _deployment.newAddress);
    }
```
As you can see in the second line code calls `_storeConstructingByteCodeHashOnAddress()` and it would set constructing bytecode hash the address(the second byte is 1), and when `_deployment.callConstructor` is false, code won't call `_constructContract()` (which sets the address's bytecode hash as constructed after calling constructor) and contract bytecode hash would stay in constructing state after the deployement.
the constructing bytecode state is there to prevent other contracts to call this contract when this contract's constructor is called. Constructing state means that "if anyone else call the contract, it will behaves like a contract being constructing (EmptyContract/EOA)". so contract won't be callable if it stays in the Constructing state. This can cause serious issues, like this scenario:
If important contracts like L1Messenger, L2EthToken, MsgValueSimulator, ... needed upgrade and their code upgraded with this deployer function(and admin didn't want to call constructor as initiating the contract again would break it), then the contract logics won't be callable by others but it would still behave like EmptyContract, the transaction won't revert and other contracts logics won't get interrupted but they don't work properly, for example user would spend funds(send to the updated contract) but because logics won't get executed the funds would be lost.

The real impact may be different based on the target contract that is being deployed by this function(when `callConstructor`=false), but in each time the contract deployment would break the contract and deployment would be faulted. the issue can happen every time and using this function to upgrade system contracts without calling constructor is common. for example imagine there is a contract, that have constructor that initialize the state. It is supposed to be called only once, for the first deployment. But protocol want to redeploy contract, that state of the contract is the same as before force deployment. So they want to skip the constructing phrase.


## Tools Used
VIM

## Recommended Mitigation Steps
even when `callConstructor` is false, and code doesn't call the constructor, code should set the address's bytecode hash to Constructed state after the deployment.