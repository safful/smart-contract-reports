## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-03

# [`characterModifier` is `uint8` but encodes `1.38e24` different Zalgo distortions.](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/277) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Tray.sol#L163
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Tray.sol#L264-L268
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L136-L174


# Vulnerability details

## Impact
Only 256 Zalgo distortions are possible, which is a miniscule fraction of the actual combinations possible.

## Proof of Concept
A Zalgo tile is defined by a letter and a modification consisting of a combination of characters above, over and below the letter. The modification consists of a selection of 1 up to 7 characters from a set of 46 characters above the letter, 0 or 1 from 5 over the letter, and 1 to 7 from 47 below the letter. Thus there is a total of $(46^1 + 46^2 + ... + 46^7)*(5^0 + 5^1)*(47^1 + 47^2 + ... + 47^7) = 1.38e24$ different Zalgo modifications possible per letter.

However, the distortions are uniquely determined by `characterModifier` which is a `uint8`. So only 256 different distortions per letter are possible.

When a tray is bought, [`_drawing` sets the tile](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Tray.sol#L163),
```solidity
function buy(uint256 _amount) external {
// ...
            trayTiledata[j] = _drawing(uint256(lastHash));
// ...
}
```
and, if luck would have it that it is a Zalgo tile, [`characterModifier` is set to a pseudo-random `uint8`](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Tray.sol#L264-L268), i.e. in the range 0..255.
```solidity
function _drawing(uint256 _seed) private pure returns (TileData memory tileData) {
// ...
            if (tileData.fontClass == 7) {
                // Set seed for Zalgo to ensure same characters will be always generated for this tile
                uint256 zalgoSeed = Utils.iteratePRNG(_seed);
                tileData.characterModifier = uint8(zalgoSeed % 256);
// ...
}
```
This `characterModifier` uniquely determines the distortion in `characterToUnicodeBytes` at [L136-L174](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L136-L174):
```solidity
} else if (_fontClass == 7) {
    // Zalgo
    uint8 asciiStartingIndex = 97;
    uint256 numAbove = (_characterModifier % 7) + 1;
    // We do not reuse the same seed for the following generations to avoid any symmetries, e.g. that 2 chars above would also always result in 2 chars below
    _characterModifier = iteratePRNG(_characterModifier);
    uint256 numMiddle = _characterModifier % 2;
    _characterModifier = iteratePRNG(_characterModifier);
    uint256 numBelow = (_characterModifier % 7) + 1;
    bytes memory character = abi.encodePacked(bytes1(asciiStartingIndex + uint8(_characterIndex)));
    for (uint256 i; i < numAbove; ++i) {
        _characterModifier = iteratePRNG(_characterModifier);
        uint256 characterIndex = (_characterModifier % ZALGO_NUM_ABOVE) * 2;
        character = abi.encodePacked(
            character,
            ZALGO_ABOVE_LETTER[characterIndex],
            ZALGO_ABOVE_LETTER[characterIndex + 1]
        );
    }
    for (uint256 i; i < numMiddle; ++i) {
        _characterModifier = iteratePRNG(_characterModifier);
        uint256 characterIndex = (_characterModifier % ZALGO_NUM_OVER) * 2;
        character = abi.encodePacked(
            character,
            ZALGO_OVER_LETTER[characterIndex],
            ZALGO_OVER_LETTER[characterIndex + 1]
        );
    }
    for (uint256 i; i < numBelow; ++i) {
        _characterModifier = iteratePRNG(_characterModifier);
        uint256 characterIndex = (_characterModifier % ZALGO_NUM_BELOW) * 2;
        character = abi.encodePacked(
            character,
            ZALGO_BELOW_LETTER[characterIndex],
            ZALGO_BELOW_LETTER[characterIndex + 1]
        );
    }
    return character;
} else {
```

Note that there is also a similar bottleneck issue with `iteratePRNG`, which is reported as a separate issue.

## Tools Used
Code inspection

## Recommended Mitigation Steps
Let `characterModifier` have a range of at least `0..1.38e24`. Since it therefore cannot be a `uint8`, simply let it be a `uint256` (or at least a `uint88`).
