## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-21

# [EIP1559 rewards received by syndicate during the period when it has no registered knots can be lost](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/376) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L218-L220
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L154-L157
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L597-L607
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L610-L627
https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L174-L197


# Vulnerability details

## Impact
When the `deRegisterKnotFromSyndicate` function is called by the DAO, the `_deRegisterKnot` function is eventually called to execute `numberOfRegisteredKnots -= 1`. It is possible that `numberOfRegisteredKnots` is reduced to 0. During the period when the syndicate has no registered knots, the EIP1559 rewards that are received by the syndicate remain in the syndicate since functions like `updateAccruedETHPerShares` do not include any logics for handling such rewards received by the syndicate. Later, when a new knot is registered and mints the derivatives, the node runner can call the `claimRewardsAsNodeRunner` function to receive half ot these rewards received by the syndicate during the period when it has no registered knots. Yet, because such rewards are received by the syndicate before the new knot mints the derivatives, the node runner should not be entitled to these rewards. Moreover, due to the issue mentioned in my other finding titled "Staking Funds vault's LP holder cannot claim EIP1559 rewards after derivatives are minted for a new BLS public key that is not the first BLS public key registered for syndicate", calling the `StakingFundsVault.claimRewards` function by the Staking Funds vault's LP holder reverts so the other half of such rewards is locked in the syndicate. Even if calling the `StakingFundsVault.claimRewards` function by the Staking Funds vault's LP holder does not revert, the Staking Funds vault's LP holder does not deserve the other half of such rewards because these rewards are received by the syndicate before the new knot mints the derivatives. Because these EIP1559 rewards received by the syndicate during the period when it has no registered knots can be unfairly sent to the node runner or remain locked in the syndicate, such rewards are lost.

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L218-L220
```solidity
    function deRegisterKnotFromSyndicate(bytes[] calldata _blsPublicKeys) external onlyDAO {
        Syndicate(payable(syndicate)).deRegisterKnots(_blsPublicKeys);
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L154-L157
```solidity
    function deRegisterKnots(bytes[] calldata _blsPublicKeys) external onlyOwner {
        updateAccruedETHPerShares();
        _deRegisterKnots(_blsPublicKeys);
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L597-L607
```solidity
    function _deRegisterKnots(bytes[] calldata _blsPublicKeys) internal {
        for (uint256 i; i < _blsPublicKeys.length; ++i) {
            bytes memory blsPublicKey = _blsPublicKeys[i];

            // Do one final snapshot of ETH owed to the collateralized SLOT owners so they can claim later
            _updateCollateralizedSlotOwnersLiabilitySnapshot(blsPublicKey);

            // Execute the business logic for de-registering the single knot
            _deRegisterKnot(blsPublicKey);
        }
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L610-L627
```solidity
    function _deRegisterKnot(bytes memory _blsPublicKey) internal {
        if (isKnotRegistered[_blsPublicKey] == false) revert KnotIsNotRegisteredWithSyndicate();
        if (isNoLongerPartOfSyndicate[_blsPublicKey] == true) revert KnotHasAlreadyBeenDeRegistered();

        // We flag that the knot is no longer part of the syndicate
        isNoLongerPartOfSyndicate[_blsPublicKey] = true;

        // For the free floating and collateralized SLOT of the knot, snapshot the accumulated ETH per share
        lastAccumulatedETHPerFreeFloatingShare[_blsPublicKey] = accumulatedETHPerFreeFloatingShare;

        // We need to reduce `totalFreeFloatingShares` in order to avoid further ETH accruing to shares of de-registered knot
        totalFreeFloatingShares -= sETHTotalStakeForKnot[_blsPublicKey];

        // Total number of registered knots with the syndicate reduces by one
        numberOfRegisteredKnots -= 1;

        emit KnotDeRegistered(_blsPublicKey);
    }
```

https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L174-L197
```solidity
    function updateAccruedETHPerShares() public {
        ...
        if (numberOfRegisteredKnots > 0) {
            ...
        } else {
            // todo - check else case for any ETH lost
        }
    }
```

## Proof of Concept
Please add the following code in `test\foundry\LiquidStakingManager.t.sol`.

1. Import `stdError` as follows.
```solidity
import { stdError } from "forge-std/Test.sol";
```

2. Add the following test. This test will pass to demonstrate the described scenario.
```solidity
    function testEIP1559RewardsReceivedBySyndicateDuringPeriodWhenItHasNoRegisteredKnotsCanBeLost() public {
        // set up users and ETH
        address nodeRunner = accountOne; vm.deal(nodeRunner, 4 ether);
        address feesAndMevUser = accountTwo; vm.deal(feesAndMevUser, 4 ether);
        address savETHUser = accountThree; vm.deal(savETHUser, 24 ether);

        // do everything from funding a validator within default LSDN to minting derivatives
        depositStakeAndMintDerivativesForDefaultNetwork(
            nodeRunner,
            feesAndMevUser,
            savETHUser,
            blsPubKeyFour
        );

        // send the syndicate some EIP1559 rewards
        uint256 eip1559Tips = 0.6743 ether;
        (bool success, ) = manager.syndicate().call{value: eip1559Tips}("");
        assertEq(success, true);

        // de-register the only knot from the syndicate to send sETH back to the smart wallet
        IERC20 sETH = IERC20(MockSlotRegistry(factory.slot()).stakeHouseShareTokens(manager.stakehouse()));
        uint256 sETHBalanceBefore = sETH.balanceOf(manager.smartWalletOfNodeRunner(nodeRunner));
        vm.startPrank(admin);
        manager.deRegisterKnotFromSyndicate(getBytesArrayFromBytes(blsPubKeyFour));
        manager.restoreFreeFloatingSharesToSmartWalletForRageQuit(
            manager.smartWalletOfNodeRunner(nodeRunner),
            getBytesArrayFromBytes(blsPubKeyFour),
            getUint256ArrayFromValues(12 ether)
        );
        vm.stopPrank();

        assertEq(
            sETH.balanceOf(manager.smartWalletOfNodeRunner(nodeRunner)) - sETHBalanceBefore,
            12 ether
        );

        vm.warp(block.timestamp + 3 hours);

        // feesAndMevUser, who is the Staking Funds vault's LP holder, can claim rewards accrued up to the point of pulling the plug
        vm.startPrank(feesAndMevUser);
        stakingFundsVault.claimRewards(feesAndMevUser, getBytesArrayFromBytes(blsPubKeyFour));
        vm.stopPrank();

        uint256 feesAndMevUserEthBalanceBefore = feesAndMevUser.balance;
        assertEq(feesAndMevUserEthBalanceBefore, (eip1559Tips / 2) - 1);

        // nodeRunner, who is the collateralized SLOT holder for blsPubKeyFour, can claim rewards accrued up to the point of pulling the plug
        vm.startPrank(nodeRunner);
        manager.claimRewardsAsNodeRunner(nodeRunner, getBytesArrayFromBytes(blsPubKeyFour));
        vm.stopPrank();
        assertEq(nodeRunner.balance, (eip1559Tips / 2));

        // more EIP1559 rewards are sent to the syndicate, which has no registered knot at this moment        
        (success, ) = manager.syndicate().call{value: eip1559Tips}("");
        assertEq(success, true);

        vm.warp(block.timestamp + 3 hours);

        // calling the claimRewards function by feesAndMevUser has no effect at this moment
        vm.startPrank(feesAndMevUser);
        stakingFundsVault.claimRewards(feesAndMevUser, getBytesArrayFromBytes(blsPubKeyFour));
        vm.stopPrank();
        assertEq(feesAndMevUser.balance, feesAndMevUserEthBalanceBefore);

        // calling the claimRewardsAsNodeRunner function by nodeRunner reverts at this moment
        vm.startPrank(nodeRunner);
        vm.expectRevert("Nothing received");
        manager.claimRewardsAsNodeRunner(nodeRunner, getBytesArrayFromBytes(blsPubKeyFour));
        vm.stopPrank();

        // however, the syndicate still holds the EIP1559 rewards received by it during the period when the only knot was de-registered
        assertEq(manager.syndicate().balance, eip1559Tips + 1);

        vm.warp(block.timestamp + 3 hours);

        vm.deal(nodeRunner, 4 ether);
        vm.deal(feesAndMevUser, 4 ether);
        vm.deal(savETHUser, 24 ether);

        // For a different BLS public key, which is blsPubKeyTwo, 
        //   do everything from funding a validator within default LSDN to minting derivatives.
        depositStakeAndMintDerivativesForDefaultNetwork(
            nodeRunner,
            feesAndMevUser,
            savETHUser,
            blsPubKeyTwo
        );

        // calling the claimRewards function by feesAndMevUser reverts at this moment
        vm.startPrank(feesAndMevUser);
        vm.expectRevert(stdError.arithmeticError);
        stakingFundsVault.claimRewards(feesAndMevUser, getBytesArrayFromBytes(blsPubKeyTwo));
        vm.stopPrank();

        // Yet, calling the claimRewardsAsNodeRunner function by nodeRunner receives half of the EIP1559 rewards
        //   received by the syndicate during the period when it has no registered knots.
        // Because such rewards are not received by the syndicate after the derivatives are minted for blsPubKeyTwo,
        //   nodeRunner does not deserve these for blsPubKeyTwo. 
        vm.startPrank(nodeRunner);
        manager.claimRewardsAsNodeRunner(nodeRunner, getBytesArrayFromBytes(blsPubKeyTwo));
        vm.stopPrank();
        assertEq(nodeRunner.balance, eip1559Tips / 2);

        // Still, half of the EIP1559 rewards that were received by the syndicate
        //   during the period when the syndicate has no registered knots is locked in the syndicate.
        assertEq(manager.syndicate().balance, eip1559Tips / 2 + 1);
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
The `else` block of the `updateAccruedETHPerShares` function (https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/syndicate/Syndicate.sol#L194-L196) can be updated to include logics that handle the EIP1559 rewards received by the syndicate during the period when it has no registered knots.