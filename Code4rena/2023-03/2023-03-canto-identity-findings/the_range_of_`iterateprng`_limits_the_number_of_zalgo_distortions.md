## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-02

# [The range of `iteratePRNG` limits the number of Zalgo distortions](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/282) 

# Lines of code

https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L136-L174
https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L256-L261


# Vulnerability details




## Impact
Only a tiny fraction of all Zalgo distortions are accessible.

## Proof of Concept
In [`characterToUnicodeBytes`, for font class `7` (i.e. Zalgo)](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L136-L174), the `_characterModifier` determines the Zalgo distortion.
The distortion is pseudo-randomly calculated by using `_characterModifier` as a seed and iteratively passing it through `iteratePRNG`, each intermediate value being used to set, in turn, each component of the distortion.
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
    // ...
```
This has the effect that the initial seed value generates a stream of values, equidistributed within their respective ranges.

A Zalgo tile is defined by a letter and a modification consisting of a combination of characters above, over and below the letter. The modification consists of a selection of 1 up to 7 characters from a set of 46 characters above the letter, 0 or 1 from 5 over the letter, and 1 to 7 from 47 below the letter. Thus there is a total of $(46^1 + 46^2 + ... + 46^7)*(5^0 + 5^1)*(47^1 + 47^2 + ... + 47^7) = 1.38e24$ different Zalgo modifications possible per letter.

However, [`iteratePRNG`](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L256-L261) modulates its return value by `2038074743`:
```solidity
function iteratePRNG(uint256 _currState) internal pure returns (uint256 iteratedState) {
    unchecked {
        iteratedState = _currState * 15485863;
        iteratedState = (iteratedState * iteratedState * iteratedState) % 2038074743;
    }
}
```
This means that after the first pass through `iteratePRNG` we only have `2038074743` different possibilities which will determine the rest of the distortion. The original `_characterModifier` is first used to set to [`uint256 numAbove = (_characterModifier % 7) + 1;`](https://github.com/code-423n4/2023-03-canto-identity/blob/077372297fc419ea7688ab62cc3fd4e8f4e24e66/canto-namespace-protocol/src/Utils.sol#L139). The remaining combinations are therefore `1.38e24 / 7 = 1.98e+23`, which is vastly greater than `2038074743`.

`characterToUnicodeBytes` can thus only return `7 * 2038074743 = 14266523201` different distortions per letter, which is only a tiny fraction of the actual number of distortions.

&nbsp;

Note that it is similarly an issue that the range of `_characterModifier` is only `0..255`, which issue is reportedly separately.

## Tools Used
Code inspection

## Recommended Mitigation Steps
`iteratePRNG` must have a range of at least 1.98e+23 different values. 