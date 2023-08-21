## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [Impossibility to change `safeCollateralRatio`](https://github.com/code-423n4/2023-06-lybra-findings/issues/882) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/configuration/LybraConfigurator.sol#L199
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraEUSDVaultBase.sol#L30
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L18


# Vulnerability details

## Impact
Because of `vaultType` variable is internal `vaultType` staticcall to vaults from the configurator will revert, so it makes it impossible to change `safeCollateralRatio`. It may be critical when market conditions will change, something happens with ETH. 

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {GovernanceTimelock} from "@lybra/governance/GovernanceTimelock.sol";
import {LybraStETHDepositVault} from "@lybra/pools/LybraStETHVault.sol";
import {Configurator} from "@lybra/configuration/LybraConfigurator.sol";
import {mockEtherPriceOracle} from "@mocks/mockEtherPriceOracle.sol";
import {mockCurve} from "@mocks/mockCurve.sol";

/* remappings used
@lybra=contracts/lybra/
@mocks=contracts/mocks/
 */
contract LybraV2SafeCollateral is Test {

    GovernanceTimelock govTimeLock;
    mockEtherPriceOracle oracle;
    mockCurve curve;
    Configurator configurator;
    LybraStETHDepositVault stETHVault;
    address owner = address(7);
    // admins && executers of GovernanceTimelock
    address[] govTimelockArr;
    IERC20 stETH = IERC20(0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84);

    function setUp() public {
        vm.startPrank(owner);
        oracle = new mockEtherPriceOracle();
        govTimelockArr.push(owner);
        govTimeLock = new GovernanceTimelock(
            1,
            govTimelockArr,
            govTimelockArr,
            owner
        );
        curve = new mockCurve();
        //  _dao , _curvePool
        configurator = new Configurator(address(govTimeLock), address(curve));

        stETHVault = new LybraStETHDepositVault(
            address(configurator),
            address(stETH),
            address(oracle)
        );
        vm.stopPrank();
    }

    function testSafeCollateral() public {
        vm.startPrank(owner);
        configurator.setSafeCollateralRatio(address(stETHVault), 165 * 1e18);
    }
    

}
```

## Tools Used
Foundry

## Recommended Mitigation Steps
Change getter function in `LybraConfigurator`:
```solidity
interface IVault {
    function getVaultType() external view returns (uint8);
}
...
...
if(IVault(pool).getVaultType() == 0) {
```
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/configuration/LybraConfigurator.sol#L29

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/configuration/LybraConfigurator.sol#L199


## Assessed type

DoS