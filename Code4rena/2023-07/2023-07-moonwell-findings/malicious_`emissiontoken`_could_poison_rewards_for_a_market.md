## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor disputed
- M-04

# [malicious `emissionToken` could poison rewards for a market](https://github.com/code-423n4/2023-07-moonwell-findings/issues/320) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L1232-L1237


# Vulnerability details

## Impact
When distributing rewards for a market, each `emissionConfig` is looped over and sent rewards for. `disburseBorrowerRewardsInternal` as an example, the same holds true for `disburseSupplierRewardsInternal`:

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MultiRewardDistributor/MultiRewardDistributor.sol#L1147-L1247
```solidity
File: MultiRewardDistributor/MultiRewardDistributor.sol

1147:    function disburseBorrowerRewardsInternal(
1148:        MToken _mToken,
1149:        address _borrower,
1150:        bool _sendTokens
1151:    ) internal {
1152:        MarketEmissionConfig[] storage configs = marketConfigs[
1153:            address(_mToken)
1154:        ];

             // ... mToken balance and borrow index

1162:        // Iterate over all market configs and update their indexes + timestamps
1163:        for (uint256 index = 0; index < configs.length; index++) {
1164:            MarketEmissionConfig storage emissionConfig = configs[index];

                 // ... index updates

1192:            if (_sendTokens) {
1193:                // Emit rewards for this token/pair
1194:                uint256 pendingRewards = sendReward( // <-- will trigger transfer on `emissionToken`
1195:                    payable(_borrower),
1196:                    emissionConfig.borrowerRewardsAccrued[_borrower],
1197:                    emissionConfig.config.emissionToken
1198:                );
1199:
1200:                emissionConfig.borrowerRewardsAccrued[
1201:                    _borrower
1202:                ] = pendingRewards;
1203:            }
1204:        }
1205:    }

...

1214:    function sendReward(
1215:        address payable _user,
1216:        uint256 _amount,
1217:        address _rewardToken
1218:    ) internal nonReentrant returns (uint256) {

             // ... short circuits breakers

1232:        uint256 currentTokenHoldings = token.balanceOf(address(this));
1233:
1234:        // Only transfer out if we have enough of a balance to cover it (otherwise just accrue without sending)
1235:        if (_amount > 0 && _amount <= currentTokenHoldings) {
1236:            // Ensure we use SafeERC20 to revert even if the reward token isn't ERC20 compliant
1237:            token.safeTransfer(_user, _amount); // <-- if this reverts all emissions fail
1238:            return 0;
1239:        } else {
                 // .. default return _amount
1245:            return _amount;
1246:        }
1247:    }
```

If one transfer reverts the whole transaction fails and no rewards will be paid out for this user. Hence if there is a malicious token that would revert on transfer it would cause no rewards to paid out. As long as there is some rewards accrued for the malicious `emissionConfig`.
The users with already unclaimed rewards for this `emissionConfig` would have their rewards permanently locked.

The `admin` (`TemporalGovernor`) of `MultiRewardsDistributor` could update the reward speed for the token to `0` but that would just prevent further damage from being done.

Upgradeable tokens aren't unusual and hence the token might seem harmless to begin with but be upgraded to a malicious implementation that reverts on transfer.

## Proof of Concept
Test in `MultiRewardDistributor.t.sol`, `MultiRewardSupplySideDistributorUnitTest`, most of the test is copied from `testSupplierHappyPath` with the addition of `MaliciousToken`:
```solidity
contract MaliciousToken {
    function balanceOf(address) public pure returns(uint256) {
        return type(uint256).max;
    }

    function transfer(address, uint256) public pure {
        revert("No transfer for you");
    }
}
```

```solidity
    function testAddMaliciousEmissionToken() public {
        uint256 startTime = 1678340000;
        vm.warp(startTime);

        MultiRewardDistributor distributor = createDistributorWithRoundValuesAndConfig(2e18, 0.5e18, 0.5e18);

        // malicious token added
        MaliciousToken token = new MaliciousToken();
        distributor._addEmissionConfig(
            mToken,
            address(this),
            address(token),
            1e18,
            1e18,
            block.timestamp + 365 days
        );

        comptroller._setRewardDistributor(distributor);

        emissionToken.allocateTo(address(distributor), 10000e18);

        vm.warp(block.timestamp + 1);

        mToken.mint(2e18);
        assertEq(MTokenInterface(mToken).totalSupply(), 2e18);

        // Wait 12345 seconds after depositing
        vm.warp(block.timestamp + 12345);

        // claim fails as the malicious token reverts on transfer
        vm.expectRevert("No transfer for you");
        comptroller.claimReward();

        // no rewards handed out
        assertEq(emissionToken.balanceOf(address(this)), 0);
    }
```

## Tools Used
Manual audit

## Recommended Mitigation Steps
Consider adding a way for `admin` to remove an `emissionConfig`.

Alternatively, the reward transfer could be wrapped in a `try`/`catch` and returning `_amount` in `catch`. Be mindful to only allow a certain amount of gas to the transfer then as otherwise the same attack works with consuming all gas available.


## Assessed type

DoS