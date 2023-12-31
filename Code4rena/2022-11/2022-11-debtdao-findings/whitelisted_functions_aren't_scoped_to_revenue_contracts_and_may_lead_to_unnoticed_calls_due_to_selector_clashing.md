## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- sponsor acknowledged
- selected for report
- M-07

# [Whitelisted functions aren't scoped to revenue contracts and may lead to unnoticed calls due to selector clashing](https://github.com/code-423n4/2022-11-debtdao-findings/issues/312) 

# Lines of code

https://github.com/debtdao/Line-of-Credit/blob/audit/code4rena-2022-11-03/contracts/utils/SpigotLib.sol#L67
https://github.com/debtdao/Line-of-Credit/blob/audit/code4rena-2022-11-03/contracts/utils/SpigotLib.sol#L14


# Vulnerability details

Whitelisted functions in the Spigot contract don't have any kind of association or validation to which revenue contract they are intended to be used. This may lead to inadvertently whitelisting a function in another revenue contract that has the same selector but a different name (signature).

## Impact

Functions in Solidity are represented by the first 4 bytes of the keccak hash of the function signature (name + argument types). It is possible (and not difficult) to find different functions that have the same selector.

In this way, a bad actor can try to use an innocent looking function that matches the selector of another function (in a second revenue contract) that has malicious intentions. The arbiter will review the innocent function, whitelist its selector, while unknowingly enabling a potential call to the malicious function, since whitelisted functions can be called on any revenue contract. 

Mining for selector clashing is feasible since selectors are 4 bytes and the search space isn't that big for current hardware.

This is similar to the attack found on proxies, documented here https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357 and here https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070

## PoC

In the following test the `collate_propagate_storage(bytes16)` function is whitelisted because it looks safe enough to the arbiter. Now, `collate_propagate_storage(bytes16) ` has the same selector as `burn(uint256)`, which allows a bad actor to call `EvilRevenueContract.burn` using the `operate` function of the Spigot.

Note: the context for this test (setup, variables and helper functions) is similar to the one found in the file `Spigot.t.sol`.

```
contract InnocentRevenueContract {
    function collate_propagate_storage(bytes16) external {
        // It's all safe here!
        console.log("Hey it's all good here");
    }
}

contract EvilRevenueContract {
    function burn(uint256) external {
        // Burn the world!
        console.log("Boom!");
    }
}

function test_WhitelistFunction_SelectorClash() public {
      vm.startPrank(owner);
      
      spigot = new Spigot(owner, treasury, operator);
      
      // Arbiter looks at InnocentRevenueContract.collate_propagate_storage and thinks it's safe to whitelist it (this is a simplified version, in a real deploy this comes from the SpigotedLine contract)
      spigot.updateWhitelistedFunction(InnocentRevenueContract.collate_propagate_storage.selector, true);
      assertTrue(spigot.isWhitelisted(InnocentRevenueContract.collate_propagate_storage.selector));
      
      // Due to selector clashing EvilRevenueContract.burn gets whitelisted too!
      assertTrue(spigot.isWhitelisted(EvilRevenueContract.burn.selector));
      
      
      EvilRevenueContract evil = new EvilRevenueContract();
      // ISpigot.Setting memory settings = ISpigot.Setting(90, claimPushPaymentFunc, transferOwnerFunc);
      // require(spigot.addSpigot(address(evil), settings), "Failed to add spigot");
      
      vm.stopPrank();
              
      // And we can call it through operate...
      vm.startPrank(operator);
      spigot.operate(address(evil), abi.encodeWithSelector(EvilRevenueContract.burn.selector, type(uint256).max));
  }
```

## Recommendation

Associate whitelisted functions to particular revenue contracts (for example, using a `mapping(address => mapping(bytes4 => bool))`) and validate that the selector for the call is enabled for that specific revenue contract in the `operate` function. 
