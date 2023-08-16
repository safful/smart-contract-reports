## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- resolved
- sponsor confirmed

# [Typo for withdawable in multiple places in Parameters.sol  ](https://github.com/code-423n4/2022-01-insure-findings/issues/244) 

# Handle

hubble


# Vulnerability details

Feel free to lower the severity of the issue to Non-critical.

## Impact
Correctness of variable name

## Proof of Concept
File : Parameters.sol

  line 39 :    mapping(address => uint256) private _withdawable;
  line 153 :         _withdawable[_address] = _target;
  line 349-352 :
        if (_withdawable[_target] == 0) {
            return _withdawable[address(0)];
        } else {
            return _withdawable[_target];

## Tools Used
Manual review

## Recommended Mitigation Steps
Change typo to _withdrawable


