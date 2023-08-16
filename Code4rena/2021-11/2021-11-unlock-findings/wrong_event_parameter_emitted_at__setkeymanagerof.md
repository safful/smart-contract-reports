## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Wrong event parameter emitted at _setKeyManagerOf](https://github.com/code-423n4/2021-11-unlock-findings/issues/84) 

# Handle

kenzo


# Vulnerability details

`_setKeyManagerOf` always emits `address(0)` as the new key manager.

## Impact
Wrong event emitted.

## Proof of Concept
The code is:
`emit KeyManagerChanged(_tokenId, address(0));`
https://github.com/code-423n4/2021-11-unlock/blob/main/smart-contracts/contracts/mixins/MixinKeys.sol#L229

## Recommended Mitigation Steps
Change line to
`emit KeyManagerChanged(_tokenId, _keyManager);`

