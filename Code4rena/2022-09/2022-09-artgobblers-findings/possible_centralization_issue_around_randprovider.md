## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- selected for report

# [Possible centralization issue around RandProvider](https://github.com/code-423n4/2022-09-artgobblers-findings/issues/327) 

# Lines of code

https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L560-L567


# Vulnerability details

## Impact
While it is very common for web3 projects to have privileged functions that can only be called by an admin address, special thought should be given to functions that can break core functionality of a project. 

One such function is ArtGobblers.upgradeRandProvider(). If this is called passing a non-compatible contract address or EOA, requesting new seeds will be bricked and as a consequence reveals will not be possible. Also the seed could be controlled using a custom contract which would allow a malicious actor to set seeds in his favor.

Naturally, the assumption that the deployer or a multisig (likely that ownership will probably be transferred to such) go rogue and perform a malicious action is unlikely to happen as they have a stake in the project (monetary and reputation wise). 

However, as this is a project that will be unleashed on launch without any further development and left to form its own ecosystem for many years to come, less centralized options should be considered.

## Proof of Concept
The function ArtGobblers.upgradeRandProvider(), allows the owner to arbitrarily pass a RandProvider with the only restriction being that there is currently no seed requested from the current RandProvider:

```js
function upgradeRandProvider(RandProvider newRandProvider) external onlyOwner {
        // Revert if waiting for seed, so we don't interrupt requests in flight.
        if (gobblerRevealsData.waitingForSeed) revert SeedPending();

        randProvider = newRandProvider; // Update the randomness provider.

        emit RandProviderUpgraded(msg.sender, newRandProvider);
    }
```

The RandProvider is the only address eligible (as well as responsible) to call ArtGobblers.acceptRandomSeed(), which is required to perform reveals of minted Gobblers:

```
function acceptRandomSeed(bytes32, uint256 randomness) external {
        // The caller must be the randomness provider, revert in the case it's not.
        if (msg.sender != address(randProvider)) revert NotRandProvider();

        // The unchecked cast to uint64 is equivalent to moduloing the randomness by 2**64.
        gobblerRevealsData.randomSeed = uint64(randomness); // 64 bits of randomness is plenty.

        gobblerRevealsData.waitingForSeed = false; // We have the seed now, open up reveals.

        emit RandomnessFulfilled(randomness);
    }
```

This could be abused with the consequences outlined under #Impact.

## Tools Used

Manual review

## Recommended Mitigation Steps
The inclusion of a voting and governance mechanism should be considered for protocol critical functions. This could for example take the form of each Gobbler representing 1 vote, with legendary Gobblers having more weight (literally) based on the amount of consumed Gobblers. 