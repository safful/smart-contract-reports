## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- H-03

# [Incorrect Reward Duration After Change in Reward Speed in MultiRewardStaking](https://github.com/code-423n4/2023-01-popcorn-findings/issues/424) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L305-L312


# Vulnerability details

## Impact
When the reward speed is changed in `MultiRewardStaking`, the new end time is calculated based off of the balance of the reward token owned by the contract. This, however, is not the same as the number of reward tokens that are left to be distributed since some of those tokens may be owed to users who have not collected their rewards yet. As a result, some users may benefit from earning rewards past the end of the intended reward period, and leaving the contract unable to pay the rewards it owes other users.

## Proof of Concept
A simple Foundry test I wrote demonstrates that the contracts fail to calculate the rewards properly after the reward speed is changed:
```// SPDX-License-Identifier: GPL-3.0
// Docgen-SOLC: 0.8.15

pragma solidity ^0.8.15;

import { Test } from "forge-std/Test.sol";
import { SafeCastLib } from "solmate/utils/SafeCastLib.sol";
import { MockERC20 } from "./utils/mocks/MockERC20.sol";
import { IMultiRewardEscrow } from "../src/interfaces/IMultiRewardEscrow.sol";
import { MultiRewardStaking, IERC20 } from "../src/utils/MultiRewardStaking.sol";
import { MultiRewardEscrow } from "../src/utils/MultiRewardEscrow.sol";

contract AuditTest is Test {
  using SafeCastLib for uint256;

  MockERC20 stakingToken;
  MockERC20 rewardToken1;
  MockERC20 rewardToken2;

  IERC20 iRewardToken1;
  IERC20 iRewardToken2;

  MultiRewardStaking staking;
  MultiRewardEscrow escrow;

  address alice = address(0xABCD);
  address bob = address(0xDCBA);
  address feeRecipient = address(0x9999);


  bytes32 constant PERMIT_TYPEHASH =
    keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

  event RewardInfoUpdate(IERC20 rewardsToken, uint160 rewardsPerSecond, uint32 rewardsEndTimestamp);
  event RewardsClaimed(address indexed user, IERC20 rewardsToken, uint256 amount, bool escrowed);

  function setUp() public {
    vm.label(alice, "alice");
    vm.label(bob, "bob");

    stakingToken = new MockERC20("Staking Token", "STKN", 18);

    rewardToken1 = new MockERC20("RewardsToken1", "RTKN1", 18);
    rewardToken2 = new MockERC20("RewardsToken2", "RTKN2", 18);
    iRewardToken1 = IERC20(address(rewardToken1));
    iRewardToken2 = IERC20(address(rewardToken2));

    escrow = new MultiRewardEscrow(address(this), feeRecipient);

    staking = new MultiRewardStaking();
    staking.initialize(IERC20(address(stakingToken)), IMultiRewardEscrow(address(escrow)), address(this));
  }

  function _addRewardToken(MockERC20 rewardsToken) internal {
    rewardsToken.mint(address(this), 10 ether);
    rewardsToken.approve(address(staking), 10 ether);

    staking.addRewardToken(IERC20(address(rewardsToken)), 0.1 ether, 10 ether, false, 0, 0, 0);
  }

  function test__endtime_after_change_reward_speed() public {
    _addRewardToken(rewardToken1);

    stakingToken.mint(alice, 1 ether);
    stakingToken.mint(bob, 1 ether);

    vm.prank(alice);
    stakingToken.approve(address(staking), 1 ether);
    vm.prank(bob);
    stakingToken.approve(address(staking), 1 ether);

    vm.prank(alice);
    staking.deposit(1 ether);

    // 50% of rewards paid out
    vm.warp(block.timestamp + 50);

    vm.prank(alice);
    staking.withdraw(1 ether);
    assertEq(staking.accruedRewards(alice, iRewardToken1), 5 ether);

    // Double Accrual (from original)
    staking.changeRewardSpeed(iRewardToken1, 0.2 ether); // Twice as fast now

    vm.prank(bob);
    staking.deposit(1 ether);

    // The remaining 50% of rewards paid out
    vm.warp(block.timestamp + 200);

    vm.prank(bob);
    staking.withdraw(1 ether);
    assertEq(staking.accruedRewards(bob, iRewardToken1), 5 ether);
  }

}
```

The output of the test demonstrates an incorrect calculation:
```
[FAIL. Reason: Assertion failed.] test__endtime_after_change_reward_speed() (gas: 558909)
Logs:
  Error: a == b not satisfied [uint]
    Expected: 5000000000000000000
      Actual: 20000000000000000000

Test result: FAILED. 0 passed; 1 failed; finished in 6.12ms
```

Notice that the amount of reward tokens given to Bob is more than the amount owned by the contract!

## Tools Used

I reproduced the bug simply by adding a test within the existing foundry project.

## Recommended Mitigation Steps

There is a nice accounting trick to make sure the remaining time is calculated correctly without needing to keep track of how much you owe to users that has not been paid out yet. I would suggest changing the vulnerable code in `changeRewardSpeed` to:

```
    uint32 prevEndTime = rewards.rewardsEndTimestamp;

    uint256 remainder = prevEndTime > block.timestamp ? (uint256(prevEndTime) - block.timestamp) * rewards.rewardsPerSecond : 0;

    uint32 rewardsEndTimestamp = _calcRewardsEnd(
      block.timestamp.safeCastTo32(),
      rewardsPerSecond,
      remainder
    );
```