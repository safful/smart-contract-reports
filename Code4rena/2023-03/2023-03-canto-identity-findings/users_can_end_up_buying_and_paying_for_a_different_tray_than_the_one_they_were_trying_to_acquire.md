## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-07

# [Users can end up buying and paying for a different Tray than the one they were trying to acquire](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/130) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/src/Tray.sol#L148-L168


# Vulnerability details

Trays and their tiles are the building blocks for fusing a Namespace NFT. The tiles in a tray are generated based on a hash of the previous mint.

Users will need specific tiles to be able to create the name they want, so it is of utter importance for them to get the trays they want with their corresponding tiles. The users should be able to precompute this, [as the documentation states](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/README.md#tray):

"The 7 tiles per tray are then generated according to a deterministic algorithm. A user can therefore precompute which trays he will get."

## Impact

Users can pay the corresponding $NOTE tokens to buy a tray with some specific tiles, buy end up obtaining a completely different one with tiles they don't expect.

This can happen due to normal usage of the platform, with some user sending a transaction with more gas. Or by a malicious user frontrunning the transaction.

On any case the user ends up with an NFT he didn't want instead of reverting the transaction, and saving their $NOTE tokens.

## Proof of Concept

There is no check on the `buy()` function to make sure that the user will actually buy the tray he wanted, thus possibly obtaining another one.

The tiles are calculated based on a hash. That hash is generated each time a new mint is produced, and it is based on the last one.

If the transaction is frontrunned, the user will get a new hash, and thus new tiles data for his tray.

```solidity
File: canto-namespace-protocol/src/Tray.sol

148:     /// @notice Buy a specifiable amount of trays
149:     /// @param _amount Amount of trays to buy
150:     function buy(uint256 _amount) external {
151:         uint256 startingTrayId = _nextTokenId();
152:         if (prelaunchMinted == type(uint256).max) {
153:             // Still in prelaunch phase
154:             if (msg.sender != owner) revert OnlyOwnerCanMintPreLaunch();
155:             if (startingTrayId + _amount - 1 > PRE_LAUNCH_MINT_CAP) revert MintExceedsPreLaunchAmount();
156:         } else {
157:             SafeTransferLib.safeTransferFrom(note, msg.sender, revenueAddress, _amount * trayPrice);
158:         }
159:         for (uint256 i; i < _amount; ++i) {
160:             TileData[TILES_PER_TRAY] memory trayTiledata;
161:             for (uint256 j; j < TILES_PER_TRAY; ++j) {
162:                 lastHash = keccak256(abi.encode(lastHash)); // @audit-info hash is calculated based on the last one
163:                 trayTiledata[j] = _drawing(uint256(lastHash)); // @audit-info tiles data is calculated based on the hash
164:             }
165:             tiles[startingTrayId + i] = trayTiledata;
166:         }
167:         _mint(msg.sender, _amount); // We do not use _safeMint on purpose here to disallow callbacks and save gas
168:     }
```

[Link to code](https://github.com/code-423n4/2023-03-canto-identity/blob/main/canto-namespace-protocol/src/Tray.sol#L148-L168)

## Tools Used

Manual review

## Recommended Mitigation Steps

Check the hash for the last minted Tray NFT, to make sure it is the same the user used to precalculate the next tiles.

On the case the transaction is frontrunned, the expected hash will be different and the transaction will revert, saving the user from spending the $NOTE tokens.

```diff
// File: canto-namespace-protocol/src/Tray.sol

    /// @notice Buy a specifiable amount of trays
    /// @param _amount Amount of trays to buy
-    function buy(uint256 _amount) external {
+    function buy(uint256 _amount, bytes32 _lastHash) external {
+       require(_lastHash == lastHash, "Hashes must be the same");
        uint256 startingTrayId = _nextTokenId();
        if (prelaunchMinted == type(uint256).max) {
            // Still in prelaunch phase
            if (msg.sender != owner) revert OnlyOwnerCanMintPreLaunch();
            if (startingTrayId + _amount - 1 > PRE_LAUNCH_MINT_CAP) revert MintExceedsPreLaunchAmount();
        } else {
            SafeTransferLib.safeTransferFrom(note, msg.sender, revenueAddress, _amount * trayPrice);
        }
        for (uint256 i; i < _amount; ++i) {
            TileData[TILES_PER_TRAY] memory trayTiledata;
            for (uint256 j; j < TILES_PER_TRAY; ++j) {
                lastHash = keccak256(abi.encode(lastHash));
                trayTiledata[j] = _drawing(uint256(lastHash));
            }
            tiles[startingTrayId + i] = trayTiledata;
        }
        _mint(msg.sender, _amount); // We do not use _safeMint on purpose here to disallow callbacks and save gas
    }
```
