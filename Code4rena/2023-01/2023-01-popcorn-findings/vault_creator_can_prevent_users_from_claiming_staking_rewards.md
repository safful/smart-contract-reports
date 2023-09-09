## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- selected for report
- sponsor confirmed
- edited-by-warden
- M-01

# [Vault creator can prevent users from claiming staking rewards](https://github.com/code-423n4/2023-01-popcorn-findings/issues/829) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardEscrow.sol#L111


# Vulnerability details

## Impact

Vault creator can prevent users from claiming rewards from the staking contract. This can boost his liquidity and lure depositors to stake vault tokens. He can present a high APY and low fee percentage which will incentivize stakers

When the staking contract is empty of stakes, he can change the settings and claim all the rewards to himself. 

## Proof of Concept

Vault creator receives management fees from two places:
1. Deposits to the vault
2. Rewards locked in escrow

Users can claim staking rewards by calling the `claimRewards` function of the staking contract:
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L179
```
  function claimRewards(address user, IERC20[] memory _rewardTokens) external accrueRewards(msg.sender, user) {
    for (uint8 i; i < _rewardTokens.length; i++) {
      uint256 rewardAmount = accruedRewards[user][_rewardTokens[i]];

      if (rewardAmount == 0) revert ZeroRewards(_rewardTokens[i]);

      EscrowInfo memory escrowInfo = escrowInfos[_rewardTokens[i]];

      if (escrowInfo.escrowPercentage > 0) {
        _lockToken(user, _rewardTokens[i], rewardAmount, escrowInfo);
        emit RewardsClaimed(user, _rewardTokens[i], rewardAmount, true);
      } else {
        _rewardTokens[i].transfer(user, rewardAmount);
        emit RewardsClaimed(user, _rewardTokens[i], rewardAmount, false);
      }

      accruedRewards[user][_rewardTokens[i]] = 0;
    }
  }
```

As can be seen above, if there is an escrow specified, percentage of rewards are sent to the escrow account through `_lockToken`:
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardStaking.sol#L201
```
  /// @notice Locks a percentage of a reward in an escrow contract. Pays out the rest to the user.
  function _lockToken(
    address user,
    IERC20 rewardToken,
    uint256 rewardAmount,
    EscrowInfo memory escrowInfo
  ) internal {
    uint256 escrowed = rewardAmount.mulDiv(uint256(escrowInfo.escrowPercentage), 1e18, Math.Rounding.Down);
    uint256 payout = rewardAmount - escrowed;

    rewardToken.safeTransfer(user, payout);
    escrow.lock(rewardToken, user, escrowed, escrowInfo.escrowDuration, escrowInfo.offset);
  }
```

`escorw.lock` will send fee to `feeRecipient` that is passed in the constructor of the escrow contract when there is a fee percentage defined. (This can be low in order to incentivize stakers)
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/utils/MultiRewardEscrow.sol#L111
```
  constructor(address _owner, address _feeRecipient) Owned(_owner) {
    feeRecipient = _feeRecipient;
  }

------

  function lock(
    IERC20 token,
    address account,
    uint256 amount,
    uint32 duration,
    uint32 offset
  ) external {
-----
    uint256 feePerc = fees[token].feePerc;
    if (feePerc > 0) {
      uint256 fee = Math.mulDiv(amount, feePerc, 1e18);

      amount -= fee;
      token.safeTransfer(feeRecipient, fee);
    }
-----
```

The issue rises when `feeRecipient` is set to the zero address (0x0). `safeTransfer` will revert and the transaction of claiming rewards will fail (all the way through the `claimRewards` on the staking contract`.

User funds will be locked in the contract. 

The vault creator can decide when is the right time to open the rewards up (maybe when nobody is invested in the vault) by changing the fee percentage to 0 using `setFees`, which bypass sending fees to feeReceipient. if the vault owner will be the only saker, he can receive all the deposited tokens back

### Foundry POC

Add the following test:
```
// SPDX-License-Identifier: GPL-3.0
// Docgen-SOLC: 0.8.15

pragma solidity ^0.8.15;

import { Test } from "forge-std/Test.sol";
import { MockERC20 } from "./utils/mocks/MockERC20.sol";
import { IMultiRewardEscrow } from "../src/interfaces/IMultiRewardEscrow.sol";
import { MultiRewardStaking, IERC20 } from "../src/utils/MultiRewardStaking.sol";
import { MultiRewardEscrow } from "../src/utils/MultiRewardEscrow.sol";

contract NoRewards is Test {
  MockERC20 stakingToken;
  MockERC20 rewardToken1;
  MockERC20 rewardToken2;
  IERC20 iRewardToken1;
  IERC20 iRewardToken2;
  MultiRewardStaking staking;
  MultiRewardEscrow escrow;

  address alice = address(0xABCD);
  address bob = address(0xDCBA);

  ///////////// ZERO ADDRESS //////////
  address feeRecipient = address(0x0);
  

  function setUp() public {
    vm.label(alice, "alice");
    vm.label(bob, "bob");

    stakingToken = new MockERC20("Staking Token", "STKN", 18);

    rewardToken1 = new MockERC20("RewardsToken1", "RTKN1", 18);
    rewardToken2 = new MockERC20("RewardsToken2", "RTKN2", 18);
    iRewardToken1 = IERC20(address(rewardToken1));
    iRewardToken2 = IERC20(address(rewardToken2));

    escrow = new MultiRewardEscrow(address(this), feeRecipient);
    IERC20[] memory tokens = new IERC20[](2);
    tokens[0] = iRewardToken1;
    tokens[1] = iRewardToken2;
    uint256[] memory fees = new uint256[](2);
    fees[0] = 1e16;
    fees[1] = 1e16;

    escrow.setFees(tokens, fees);
    staking = new MultiRewardStaking();
    staking.initialize(IERC20(address(stakingToken)), IMultiRewardEscrow(address(escrow)), address(this));
  }

  function _addRewardToken(MockERC20 rewardsToken) internal {
    rewardsToken.mint(address(this), 10 ether);
    rewardsToken.approve(address(staking), 10 ether);
    staking.addRewardToken(IERC20(address(rewardsToken)), 0.1 ether, 10 ether, true, 30, 100, 0);
  }

  function test__no_rewards() public {
    // Prepare array for `claimRewards`
    IERC20[] memory rewardsTokenKeys = new IERC20[](1);
    rewardsTokenKeys[0] = iRewardToken1;

    _addRewardToken(rewardToken1);
    stakingToken.mint(alice, 5 ether);

    vm.startPrank(alice);
    stakingToken.approve(address(staking), 5 ether);
    staking.deposit(1 ether);

    // 10% of rewards paid out
    vm.warp(block.timestamp + 10);

    uint256 callTimestamp = block.timestamp;
    staking.claimRewards(alice, rewardsTokenKeys);
  }
}

```

test `test__no_rewards`
## Tools Used

VS Code, Foundry

## Recommended Mitigation Steps

Validate in the initialization of the staking contracts that escrow.feeRecipient is not the zero address