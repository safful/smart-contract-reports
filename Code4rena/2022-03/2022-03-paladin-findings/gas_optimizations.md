## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas Optimizations](https://github.com/code-423n4/2022-03-paladin-findings/issues/34) 

## Don't Initialize Variables with Default Value

### Impact
Issue Information: [G001](https://github.com/byterocket/c4-common-issues/blob/main/0-Gas-Optimizations.md#g001---dont-initialize-variables-with-default-value)

### Findings:
```
HolyPaladinToken.sol::516 => uint256 low = 0;
HolyPaladinToken.sol::688 => uint256 low = 0;
HolyPaladinToken.sol::796 => uint256 userLockedBalance = 0;
HolyPaladinToken.sol::807 => uint256 lockingRewards = 0;
HolyPaladinToken.sol::940 => uint256 low = 0;
HolyPaladinToken.sol::972 => uint256 low = 0;
HolyPaladinToken.sol::1004 => uint256 low = 0;
```
### Tools used
[c4udit](https://github.com/byterocket/c4udit)