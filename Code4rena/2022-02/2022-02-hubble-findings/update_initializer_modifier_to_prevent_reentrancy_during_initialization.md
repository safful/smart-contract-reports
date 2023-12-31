## Tags

- bug
- 3 (High Risk)
- resolved
- sponsor confirmed

# [Update initializer modifier to prevent reentrancy during initialization](https://github.com/code-423n4/2022-02-hubble-findings/issues/81) 

# Lines of code

https://github.com/code-423n4/2022-02-hubble/blob/main/package.json#L17
https://github.com/code-423n4/2022-02-hubble/blob/main/contracts/legos/Governable.sol#L5
https://github.com/code-423n4/2022-02-hubble/blob/main/contracts/legos/Governable.sol#L24


# Vulnerability details

## Impact

While Governable.sol is out of scope, I figured this issue would still be fair game.

The solution uses: `"@openzeppelin/contracts": "4.2.0"`.
This dependency has a known high severity vulnerability: https://security.snyk.io/vuln/SNYK-JS-OPENZEPPELINCONTRACTS-2320176
Which makes this contract vulnerable:
```
File: Governable.sol
05: import { Initializable } from "@openzeppelin/contracts/proxy/utils/Initializable.sol";
...
24: contract Governable is VanillaGovernable, Initializable {}
```

This contract is inherited at multiple places:
```
contracts/AMM.sol:
  11: contract AMM is IAMM, Governable {

contracts/InsuranceFund.sol:
  13: contract InsuranceFund is VanillaGovernable, ERC20Upgradeable {

contracts/Oracle.sol:
  11: contract Oracle is Governable {

contracts/legos/HubbleBase.sol:
  15: contract HubbleBase is Governable, Pausable, ERC2771Context {

contracts/ClearingHouse.sol:
  11: contract ClearingHouse is IClearingHouse, HubbleBase {

contracts/MarginAccount.sol:
  25: contract MarginAccount is IMarginAccount, HubbleBase {
```

 ìnitializer()` is used here:
```
contracts/AMM.sol:
  99:     ) external initializer {

contracts/ClearingHouse.sol:
  44:     ) external initializer {

contracts/MarginAccount.sol:
  124:     ) external initializer {

contracts/Oracle.sol:
  20:     function initialize(address _governance) external initializer {

```

## Recommended Mitigation Steps
Upgrade `@openzeppelin/contracts` to version 4.4.1 or higher.

