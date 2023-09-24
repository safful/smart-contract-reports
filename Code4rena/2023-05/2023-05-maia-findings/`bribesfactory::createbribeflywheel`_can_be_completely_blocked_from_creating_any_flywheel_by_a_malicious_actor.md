## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-29

# [`BribesFactory::createBribeFlywheel` can be completely blocked from creating any Flywheel by a malicious actor](https://github.com/code-423n4/2023-05-maia-findings/issues/362) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L125-L129


# Vulnerability details

## Impact
A malicious actor can completely block the creation of any bribe flywheel that is created via `BribesFactory::createBribeFlywheel` because of the way the `FlywheelBribeRewards` parameter is set.
Initially it is set [to the zero address in its constructor](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BribesFactory.sol#L84) and then reset to [a different address via the `FlywheelCore::setFlywheelRewards` call](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BribesFactory.sol#L96) (in the same transaction). 

```Solidity
    function createBribeFlywheel(address bribeToken) public {
        // ...

        FlywheelCore flywheel = new FlywheelCore(
            bribeToken,
            FlywheelBribeRewards(address(0)),
            flywheelGaugeWeightBooster,
            address(this)
        );

        // ...

        flywheel.setFlywheelRewards(address(new FlywheelBribeRewards(flywheel, rewardsCycleLength)));
        
        // ...
    }
```

The `FlywheelCore::setFlywheelRewards` function verifies if the current `flywheelRewards` address has any balance of the provided reward token and, if so, [transfers it to the new `flywheelRewards` address](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L125-L129).

```Solidity
    function setFlywheelRewards(address newFlywheelRewards) external onlyOwner {
        uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
        if (oldRewardBalance > 0) {
            rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
        }
```

The issue is that `FlywheelCore::setFlywheelRewards` does not check if the current `FlywheelCore::flywheelRewards` address is 0 and thus attempts to transfer from 0 address if that address has any reward token in it. 
A malicious actor can simply send 1 wei of `rewardToken` to the zero address and all `BribesFactory::createBribeFlywheel` will fail because of the attempted transfer of tokens from the 0 address.

This is also an issue for any 3rd party project, that wishes to use MaiaDAO's `BribesFactory` implementation, and uses a burnable reward token because most likely normal users (non-malicious) have already burned (sent to zero address) tokens so the creating of bribe factories would fail by default.

Another observation is that, because all MaiaDAO project tokens use the Solmate ERC20 implementation they [all can transfer to 0 (burn)](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L76-L83) which makes this scenario real even if using project tokens as reward tokens.

## Proof of Concept

A coded POC follows, add it to `test\gauges\factories\BribesFactoryTest.t.sol`:

```Solidity
    import {stdError} from "forge-std/Test.sol";

    function testDosCreateBribeFlywheel() public {
        MockERC20 bribeToken3 = new MockERC20("Bribe Token3", "BRIBE3", 18);
        bribeToken3.mint(address(this), 1000);
        
        // transfer 1 wei to zero address (or "burn" on other contracts)
        bribeToken3.transfer(address(0), 1);
        assertEq(bribeToken3.balanceOf(address(0)), 1);
                
        // hevm.expectRevert(stdError.arithmeticError); // for some reason this does not work, foundry error        
        // function reverts regardless with "Arithmetic over/underflow" because the way Solmate ERC20::transferFrom is implemented
        factory.createBribeFlywheel(address(bribeToken3)); 
    }
```

Observation: because the `MockERC20` contract uses Solmate ERC20 implementation, the error is `"Arithmetic over/underflow"` since `address(0)` did not pre-approve the token swap (evidently).

## Tools Used

Manual analysis

## Recommended Mitigation Steps

- if project tokens are to be used as reward tokens, consider using OpenZeppelin ERC20 implementation (as [it does not allow transfer to 0 address](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L225-L233) if burn is not intended) or add checks to all project token contracts that transfer `to` argument must never be `address(0)`.
- modify `FlywheelCore::setFlywheelRewards` to not attempt any token transfer if previous `flywheelRewards` is `address(0)`. Example implementation:

```diff
diff --git a/src/rewards/base/FlywheelCore.sol b/src/rewards/base/FlywheelCore.sol
index 308b804..eaa0093 100644
--- a/src/rewards/base/FlywheelCore.sol
+++ b/src/rewards/base/FlywheelCore.sol
@@ -123,9 +123,11 @@ abstract contract FlywheelCore is Ownable, IFlywheelCore {
 
     /// @inheritdoc IFlywheelCore
     function setFlywheelRewards(address newFlywheelRewards) external onlyOwner {
-        uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
-        if (oldRewardBalance > 0) {
-            rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
+        if (address(flywheelRewards) != address(0)) {
+            uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
+            if (oldRewardBalance > 0) {
+                rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
+            }
         }
 
         flywheelRewards = newFlywheelRewards;

```


## Assessed type

DoS