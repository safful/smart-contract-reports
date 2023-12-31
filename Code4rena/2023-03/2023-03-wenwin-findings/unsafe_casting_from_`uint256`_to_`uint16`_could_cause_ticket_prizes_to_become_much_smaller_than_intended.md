## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-05

# [Unsafe casting from `uint256` to `uint16` could cause ticket prizes to become much smaller than intended](https://github.com/code-423n4/2023-03-wenwin-findings/issues/245) 

# Lines of code

https://github.com/code-423n4/2023-03-wenwin/blob/main/src/LotterySetup.sol#L164-L176


# Vulnerability details

## Vulnerability Details

In `LotterySetup.sol`, the `packFixedRewards()` function packs a `uint256` array into a `uint256` through bitwise arithmetic:

[`LotterySetup.sol#L164-L176`](https://github.com/code-423n4/2023-03-wenwin/blob/main/src/LotterySetup.sol#L164-L176):
```solidity
function packFixedRewards(uint256[] memory rewards) private view returns (uint256 packed) {
    if (rewards.length != (selectionSize) || rewards[0] != 0) {
        revert InvalidFixedRewardSetup();
    }
    uint256 divisor = 10 ** (IERC20Metadata(address(rewardToken)).decimals() - 1);
    for (uint8 winTier = 1; winTier < selectionSize; ++winTier) {
        uint16 reward = uint16(rewards[winTier] / divisor);
        if ((rewards[winTier] % divisor) != 0) {
            revert InvalidFixedRewardSetup();
        }
        packed |= uint256(reward) << (winTier * 16);
    }
}
```

The `rewards[]` parameter stores the prize amount per each `winTier`, where `winTier` is the number of matching numbers a ticket has. `packFixedRewards()` is used when the lottery is first initialized to store the prize for each non-jackpot `winTier`.

The vulnerability lies in the following line:

```solidity
uint16 reward = uint16(rewards[winTier] / divisor);
```

It casts `rewards[winTier] / divisor`, which is a `uint256`, to a `uint16`. If `rewards[winTier] / divisor` is larger than `2 ** 16 - 1`, the unsafe cast will only keep its rightmost bits, causing the result to be much smaller than defined in `rewards[]`. 

As `divisor` is defined as `10 ** (tokenDecimals - 1)`, the upperbound of `rewards[winTier]` evaluates to `6553.5 * 10 ** tokenDecimals`. This means that the prize of any `winTier` must not be larger than 6553.5 tokens, otherwise the unsafe cast causes it to become smaller than expected.

## Impact

If a deployer is unaware of this upper limit, he could deploy the lottery with ticket prizes larger than 6553.5 tokens, causing non-jackpot ticket prizes to become significantly smaller. The likelihood of this occuring is increased as:

1. The upper limit is not mentioned anywhere in the documentation.
2. The upper limit is not immediately obvious when looking at the code.
   
This upper limit also restricts the protocol from using low price tokens. For example, if the protocol uses SHIB ($0.00001087 per token), the highest possible prize with 6553.5 tokens is worth only $0.071236545.

## Proof of Concept

If the lottery is initialized with `rewards = [0, 6500, 7000]`, the prize for each `winTier` would become the following:

| `winTier` | Token Amount (in tokenDecimals) |
| --------- | ------------------------------- |
| 0         | 0                               |
| 1         | 6500                            |
| 2         | 446                             |


The prize for `winTier = 2` can be derived as such:

```
(tokenAmount * 10) & type(uint16).max = (7000 * 10) & (2 ** 16 - 1) = 4464
4464 / 10 = 446
```

The following test demonstrates the above:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/LotterySetup.sol";
import "./TestToken.sol";

contract RewardUnsafeCastTest is Test {
    ERC20 public rewardToken;

    uint8 public constant SELECTION_SIZE = 3;
    uint8 public constant SELECTION_MAX = 10;

    function setUp() public {
        rewardToken = new TestToken();
    }

    function testRewardIsSmallerThanExpected() public {
        // Get 1 token unit
        uint256 tokenUnit = 10 ** rewardToken.decimals();

        // Define fixedRewards as [0, 6500, 7000]
        uint256[] memory fixedRewards = new uint256[](SELECTION_SIZE);
        fixedRewards[1] = 6500 * tokenUnit;
        fixedRewards[2] = 7000 * tokenUnit;

        // Initialize LotterySetup contract
        LotterySetup lotterySetup = new LotterySetup(
            LotterySetupParams(
                rewardToken,
                LotteryDrawSchedule(block.timestamp + 2*100, 100, 60),
                5 ether,
                SELECTION_SIZE,
                SELECTION_MAX,
                38e16,
                fixedRewards
            )
        );

        // Reward for winTier 1 is 6500
        assertEq(lotterySetup.fixedReward(1) / tokenUnit, 6500);

        // Reward for winTier 2 is 446 instead of 7000
        assertEq(lotterySetup.fixedReward(2) / tokenUnit, 446);
    }
}
```

## Recommended Mitigation

Consider storing the prize amount of each `winTier` in a mapping instead of packing them into a `uint256` using bitwise arithmetic. This approach removes the upper limit (6553.5) and lower limit (0.1) for prizes, which would allow the protocol to use tokens with extremely high or low prices.

Alternatively, check if `rewards[winTier] > type(uint256).max` and revert if so. This can be done through OpenZeppelin's [SafeCast](https://docs.openzeppelin.com/contracts/3.x/api/utils#SafeCast-toUint16-uint256-).