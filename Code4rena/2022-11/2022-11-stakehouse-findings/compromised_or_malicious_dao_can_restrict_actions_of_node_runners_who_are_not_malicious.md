## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-26

# [Compromised or malicious DAO can restrict actions of node runners who are not malicious](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/383) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LSDNFactory.sol#L73-L102
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L239-L246
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L308-L321
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L356-L377
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L326-L350


# Vulnerability details

## Impact
When calling the `deployNewLiquidStakingDerivativeNetwork` function, `_dao` is not required to be an address that corresponds to a governance contract. This is also confirmed by the code walkthrough at https://www.youtube.com/watch?v=7UHDUA9l6Ek&t=650s, which mentions that `_dao` can correspond to an address of a single user. Especially when the DAO is set to be an EOA address, it is possible that its private key becomes compromised. Moreover, because the `updateDAOAddress` function lacks a two step procedure for transferring the DAO's role, it is possible that the DAO is set to an uncontrolled address, which can be malicious. When the DAO becomes compromised or malicious, the actions of the node runners, who are not malicious, can be restricted at the DAO's will, such as by calling functions like `rotateEOARepresentativeOfNodeRunner` and `rotateNodeRunnerOfSmartWallet`. For example, a compromised DAO can call the `rotateNodeRunnerOfSmartWallet` function to transfer a smart wallet from a node runner, who is not malicious at all, to a colluded party. Afterwards, the affected node runner is banned from many interactions with the protocol and can no longer call, for instance, the `withdrawETHForKnot` function for withdrawing ETH from the corresponding smart wallet. Hence, a compromised or malicious DAO can cause severe consequences, including ETH losses.

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LSDNFactory.sol#L73-L102
```solidity
    function deployNewLiquidStakingDerivativeNetwork(
        address _dao,
        uint256 _optionalCommission,
        bool _deployOptionalHouseGatekeeper,
        string calldata _stakehouseTicker
    ) public returns (address) {

        // Clone a new liquid staking manager instance
        address newInstance = Clones.clone(liquidStakingManagerImplementation);
        ILiquidStakingManager(newInstance).init(
            _dao,
            ...
        );

        ...
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L239-L246
```solidity
    function updateDAOAddress(address _newAddress) external onlyDAO {
        require(_newAddress != address(0), "Zero address");
        require(_newAddress != dao, "Same address");

        emit UpdateDAOAddress(dao, _newAddress);

        dao = _newAddress;
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L308-L321
```solidity
    function rotateEOARepresentativeOfNodeRunner(address _nodeRunner, address _newRepresentative) external onlyDAO {
        require(_newRepresentative != address(0), "Zero address");

        address smartWallet = smartWalletOfNodeRunner[_nodeRunner];
        require(smartWallet != address(0), "No smart wallet");
        require(stakedKnotsOfSmartWallet[smartWallet] == 0, "Not all KNOTs are minted");
        require(smartWalletRepresentative[smartWallet] != _newRepresentative, "Invalid rotation to same EOA");

        // unauthorize old representative
        _authorizeRepresentative(smartWallet, smartWalletRepresentative[smartWallet], false);

        // authorize new representative
        _authorizeRepresentative(smartWallet, _newRepresentative, true);
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L356-L377
```solidity
    function rotateNodeRunnerOfSmartWallet(address _current, address _new, bool _wasPreviousNodeRunnerMalicious) external {
        require(_new != address(0) && _current != _new, "New is zero or current");

        address wallet = smartWalletOfNodeRunner[_current];
        require(wallet != address(0), "Wallet does not exist");
        require(_current == msg.sender || dao == msg.sender, "Not current owner or DAO");

        address newRunnerCurrentWallet = smartWalletOfNodeRunner[_new];
        require(newRunnerCurrentWallet == address(0), "New runner has a wallet");

        smartWalletOfNodeRunner[_new] = wallet;
        nodeRunnerOfSmartWallet[wallet] = _new;

        delete smartWalletOfNodeRunner[_current];

        if (msg.sender == dao && _wasPreviousNodeRunnerMalicious) {
            bannedNodeRunners[_current] = true;
            emit NodeRunnerBanned(_current);
        }

        emit NodeRunnerOfSmartWalletRotated(wallet, _current, _new);
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L326-L350
```solidity
    function withdrawETHForKnot(address _recipient, bytes calldata _blsPublicKeyOfKnot) external {
        ...

        address associatedSmartWallet = smartWalletOfKnot[_blsPublicKeyOfKnot];
        require(smartWalletOfNodeRunner[msg.sender] == associatedSmartWallet, "Not the node runner for the smart wallet ");
        require(isNodeRunnerBanned(nodeRunnerOfSmartWallet[associatedSmartWallet]) == false, "Node runner is banned from LSD network");
        ...
    }
```

## Proof of Concept
Please add the following test in `test\foundry\LSDNFactory.t.sol`. This test will pass to demonstrate the described scenario.
```solidity
    function testCompromisedDaoCanRestrictActionsOfNodeRunnersWhoAreNotMalicious() public {
        vm.prank(address(factory));
        manager.updateDAOAddress(admin);

        uint256 nodeStakeAmount = 4 ether;
        address nodeRunner = accountOne;
        vm.deal(nodeRunner, nodeStakeAmount);

        address eoaRepresentative = accountTwo;

        vm.prank(nodeRunner);
        manager.registerBLSPublicKeys{value: nodeStakeAmount}(
            getBytesArrayFromBytes(blsPubKeyOne),
            getBytesArrayFromBytes(blsPubKeyOne),
            eoaRepresentative
        );

        // Simulate a situation where admin, who is the dao at this moment, is compromised.
        // Although nodeRunner is not malicious,
        //   the compromised admin can call the rotateNodeRunnerOfSmartWallet function to assign nodeRunner's smart wallet to a colluded party.
        vm.prank(admin);
        manager.rotateNodeRunnerOfSmartWallet(nodeRunner, accountThree, true);

        // nodeRunner is blocked from other interactions with the protocol since it is now banned unfairly
        assertEq(manager.bannedNodeRunners(accountOne), true);

        // for example, nodeRunner is no longer able to call the withdrawETHForKnot function
        vm.prank(nodeRunner);
        vm.expectRevert("Not the node runner for the smart wallet ");
        manager.withdrawETHForKnot(nodeRunner, blsPubKeyOne);
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
When calling the `deployNewLiquidStakingDerivativeNetwork` function, instead of explicitly setting the DAO's address, a configurable governance contract, which can have features like voting and timelock, can be deployed and used as the DAO.