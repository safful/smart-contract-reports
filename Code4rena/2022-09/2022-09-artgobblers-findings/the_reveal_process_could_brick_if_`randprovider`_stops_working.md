## Tags

- bug
- 2 (Med Risk)
- edited-by-warden
- selected for report

# [The reveal process could brick if `randProvider` stops working](https://github.com/code-423n4/2022-09-artgobblers-findings/issues/153) 

# Lines of code

https://github.com/artgobblers/art-gobblers/blob/master/src/ArtGobblers.sol#L562


# Vulnerability details

It could become impossible to reveal gobblers which leads any new minted gobbler to have an `emissionMultiple` of `0` forever, preventing them from minting any goo.

## Proof of concept

In `ArtGobblers.sol` calling `requestRandomSeed()` sets `gobblerRevealsData.waitingForSeed = true`, which makes both `revealGobblers()` and `upgradeRandProvider()` revert.

The only way to set `gobblerRevealsData.waitingForSeed = false` is by calling `acceptRandomSeed()` which can only be called by the `randProvider` itself.

This means that if `randProvider` stops working and `requestRandomSeed()` is called (which sets `gobblerRevealsData.waitingForSeed = true`) there is no way for `upgradeRandProvider()` and `revealGobblers()` to be ever called again.

Copy the test in `RandProvider.t.sol` and run it with `forge test -m testRandomnessBrick`:

```javascript
function testRandomnessBrick() public {
    vm.warp(block.timestamp + 1 days);
    mintGobblerToAddress(users[0], 1);

    bytes32 requestId = gobblers.requestRandomSeed();

    /*
        At this point we should have
        vrfCoordinator.callBackWithRandomness(requestId, randomness, address(randProvider));

        But that doesn't happen because we are assuming
        randProvider stopped responding
    /*
    
    /*
        Can't reveal gobblers
    */
    vm.expectRevert(ArtGobblers.SeedPending.selector);
    gobblers.revealGobblers(1);

    /*
        Can't update provider
    */
    RandProvider newRandProvider = new ChainlinkV1RandProvider(
        ArtGobblers(address(gobblers)),
        address(vrfCoordinator),
        address(linkToken),
        keyHash,
        fee
    );

    vm.expectRevert(ArtGobblers.SeedPending.selector);
    gobblers.upgradeRandProvider(newRandProvider);
}
```
## Mitigation Steps

A potential fix is to reset some variables in `upgradeRandProvider` instead of reverting:
```javascript
function upgradeRandProvider(RandProvider newRandProvider) external onlyOwner {
    if (gobblerRevealsData.waitingForSeed) {
        gobblerRevealsData.waitingForSeed = false; 
        gobblerRevealsData.toBeRevealed = 0;
        gobblerRevealsData.nextRevealTimestamp = uint64(nextRevealTimestamp - 1 days);
    }

    randProvider = newRandProvider; // Update the randomness provider.

    emit RandProviderUpgraded(msg.sender, newRandProvider);
}
```

This gives a little extra powers to the owner, but I don't think it's a big deal considering that he can just plug in any provider he wants anyway.
