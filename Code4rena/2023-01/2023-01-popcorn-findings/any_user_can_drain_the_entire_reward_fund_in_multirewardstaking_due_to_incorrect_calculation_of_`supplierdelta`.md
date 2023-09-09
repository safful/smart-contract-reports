## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-01

# [Any user can drain the entire reward fund in MultiRewardStaking due to incorrect calculation of `supplierDelta`](https://github.com/code-423n4/2023-01-popcorn-findings/issues/791) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L406
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L427
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L274


# Vulnerability details

## Impact

Reward `deltaIndex` in `_accrueRewards()` is multiplied by `10**decimals()` but eventually divided by `rewards.ONE` (which is equal to `10**IERC20Metadata(address(rewardToken)).decimals()`) in `_accrueUser()`. 

If the number of decimals in MultiRewardEscrow share token differs from the number of decimals in the reward token, then all rewards are multipled by `10 ** (decimals() - rewardToken.decimals())`.

Therefore, for example, if an admin adds USDT as the reward token with decimals=6, it will result in the reward for any user to be multiplied by ``10**(18-6) = 1000000000000` on the next block. This will at best lead to a DOS where no one will be able to withdraw funds. But at worst, users will drain the entire reward fund due to inflated calculations in the next block.


## Proof of Concept


Put the following test in `./test/` folder and run with `forge test --mc DecimalMismatchTest`. The test fails because of incorrect `supplierDelta` calculations:
```solidity
// SPDX-License-Identifier: GPL-3.0
// Docgen-SOLC: 0.8.15

pragma solidity ^0.8.15;

import { Test } from "forge-std/Test.sol";
import { SafeCastLib } from "solmate/utils/SafeCastLib.sol";
import { MockERC20 } from "./utils/mocks/MockERC20.sol";
import { IMultiRewardEscrow } from "../src/interfaces/IMultiRewardEscrow.sol";
import { MultiRewardStaking, IERC20 } from "../src/utils/MultiRewardStaking.sol";
import { MultiRewardEscrow } from "../src/utils/MultiRewardEscrow.sol";

contract DecimalMismatchTest is Test {
  using SafeCastLib for uint256;

  MockERC20 stakingToken;
  MockERC20 rewardToken;
  MultiRewardStaking staking;
  MultiRewardEscrow escrow;

  address alice = address(0xABCD);
  address bob = address(0xDCBA);
  address feeRecipient = address(0x9999);

  function setUp() public {
    vm.label(alice, "alice");
    vm.label(bob, "bob");

    // staking token has 18 decimals
    stakingToken = new MockERC20("Staking Token", "STKN", 18);

    // reward token has 6 decimals (for example USDT)
    rewardToken = new MockERC20("RewardsToken1", "RTKN1", 6);

    escrow = new MultiRewardEscrow(address(this), feeRecipient);

    staking = new MultiRewardStaking();
    staking.initialize(IERC20(address(stakingToken)), IMultiRewardEscrow(address(escrow)), address(this));
    
    rewardToken.mint(address(this), 1000 ether);
    rewardToken.approve(address(staking), 1000 ether);

    staking.addRewardToken(
      // rewardToken
      IERC20(address(rewardToken)), 
    
      // rewardsPerSecond
      1e10, 

      // amount
      1e18, 

      // useEscrow
      false, 

      // escrowPercentage
      0, 

      // escrowDuration
      0, 

      // offset
      0
    );

  }

  function testWrongSupplierDelta() public {
    stakingToken.mint(address(bob), 1);
    
    vm.prank(bob);
    stakingToken.approve(address(staking), 1);
    
    vm.prank(bob);
    staking.deposit(1);

    assert (staking.balanceOf(bob) == 1);

    vm.warp(block.timestamp + 1);

    IERC20[] memory a = new IERC20[](1);
    a[0] = IERC20(address(rewardToken));

    vm.prank(bob);

    // 1 second elapsed, so Bob must get a little reward
    // but instead this will REVERT with "ERC20: transfer amount exceeds balance"
    // because the `supplierDelta` is computed incorrect and becomes too large
    staking.claimRewards(bob, a);
  }
}
```


## Tools Used

Manual analysis


## Recommended Mitigation Steps

Use the same number of decimals when calculating `deltaIndex` and `supplierDelta`.
