## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- disagree with severity
- resolved

# [Market-specific pause is not checked for sponsor](https://github.com/code-423n4/2021-06-realitycards-findings/issues/145) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

The treasury only checks its `globalPause` field but does not check its market-specific `marketPaused` field for `Treasury.sponsor`.
A paused market contract can therefore still deposit as a sponsor using `Market.sponsor`

## Impact

The market-specific pause does not work correctly.

## Recommended Mitigation Steps

Add checks for `marketPaused` in the Treasury for `sponsor`.


