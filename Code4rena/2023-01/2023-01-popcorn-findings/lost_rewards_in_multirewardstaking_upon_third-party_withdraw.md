## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- H-06

# [Lost Rewards in MultiRewardStaking Upon Third-Party Withdraw](https://github.com/code-423n4/2023-01-popcorn-findings/issues/386) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L127


# Vulnerability details

## Impact

Affected contract: `MultiRewardStaking`

When assets are withdrawn for user `Alice` by an approved user `Bob` to a receiver that is not `Alice`, the rewards are never accrued and the resulting **staking rewards are lost forever**. This is because `accrueRewards` is called on `caller` and `receiver` but never `owner`.

Third-party withdrawals are allowed by the fact that `withdraw(uint256, address, address)` exists in `ERC4626Upgradeable` and is never overwritten by a method with the same signature. Protocols composing with Popcorn will assume by the nature of this contract being an `ERC4626` that this method is safe to use when it in fact costs the user significantly. 

## Proof of Concept
I created a test to reproduce this bug. When I included the below code within `MultiRewardStaking.t.sol` it passed, meaning Alice and Bob both had no rewards to claim by the end:

```
function test__withdraw_bug() public {
    // Add a reward token
    _addRewardToken(rewardToken1); // adds at 0.1 per second

    // Make a deposit for Alice
    stakingToken.mint(alice, 1 ether);
    vm.prank(alice);
    stakingToken.approve(address(staking), 1 ether);
    assertEq(stakingToken.allowance(alice, address(staking)), 1 ether);

    assertEq(staking.balanceOf(alice), 0);
    vm.prank(alice);
    staking.deposit(1 ether);
    assertEq(staking.balanceOf(alice), 1 ether);

    // Move 10 seconds into the future
    vm.warp(block.timestamp + 10); // 1 ether should be owed to Alice in rewards

    // Approve Bob for withdrawal
    vm.prank(alice);
    staking.approve(bob, 1 ether);

    // Bob withdraws to himself
    vm.prank(bob);
    staking.withdraw(1 ether, bob, alice);
    assertEq(staking.balanceOf(alice), 0);
    assertEq(stakingToken.balanceOf(bob), 1 ether);

    IERC20[] memory rewardsTokenKeys = new IERC20[](1);
    rewardsTokenKeys[0] = iRewardToken1;

    // Alice has no rewards to claim
    vm.prank(alice);
    vm.expectRevert(abi.encodeWithSelector(MultiRewardStaking.ZeroRewards.selector, iRewardToken1));
    staking.claimRewards(alice, rewardsTokenKeys);

    // Bob has no rewards to claim
    vm.prank(bob);
    vm.expectRevert(abi.encodeWithSelector(MultiRewardStaking.ZeroRewards.selector, iRewardToken1));
    staking.claimRewards(bob, rewardsTokenKeys);
  }
```

One can similarly create a test that doesn't expect the calls at the end to revert and that test will fail.

## Tools Used

I reproduced the bug simply by adding a test within the existing foundry project.

## Recommended Mitigation Steps

1) Fix the code by changing [this line of code](https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L127) in `_withdraw` to instead call `_accrueRewards(owner, receiver)`. It is okay to not accrue the rewards on `caller` since the caller neither gains nor loses staked tokens.

2) Add a similar test as above in `MultiRewardStaking.t.sol` that **will fail** if Alice is unable to withdraw `1 ether` of rewards in the end.