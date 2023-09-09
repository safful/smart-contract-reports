## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- selected for report
- sponsor confirmed
- edited-by-warden
- M-26

# [**Harvest()** may not be executed when changing a **Vault** adapter](https://github.com/code-423n4/2023-01-popcorn-findings/issues/284) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L594-L613
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L162


# Vulnerability details

## Impact
Changing the adapter (that uses a strategy) of an already credited vault can result in a loss of user funds.

## Proof of Concept
**Senario :** 
- A **Vault** owner want to change the underlying **Adapter** 
- Owner call the **proposeAdapter()** and then **changeAdapter()** that will call the **redeem()** adapter function :
  -  https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L193-L235
- Here the goal is to empty the **Strategy** and **Adapter** underlying contracts of the  **Vault** to make a safe adapter change.

Issue senario **1** : the **harvest()** function is in cooldown. 

Issue senario **2** : the **harvest()** function revert.

  
In both case, the **harvest()** fonction will not execute. The adapter will change without harvesting from the **Strategy** causing the loss of unclaimed rewards.

## Tools Used
None

## Recommended Mitigation Steps

Change the **harvest()** function from :
``` js
function harvest() public takeFees {
    if (
        address(strategy) != address(0) &&
        ((lastHarvest + harvestCooldown) < block.timestamp)
    ) {
        // solhint-disable
        address(strategy).delegatecall(
            abi.encodeWithSignature("harvest()")
        );
    }

    emit Harvested();
}
```
to : 
``` js
function harvest() public takeFees {
    if (
        address(strategy) == address(0) ||
        ((lastHarvest + harvestCooldown) > block.timestamp)
    ) {
        revert();  // Fixing the "Issue senario 1"
    }

    (bool success, ) = address(strategy).delegatecall(
        abi.encodeWithSignature("harvest()")
    );

    if (!success) revert(); // Fixing the "Issue senario 2"

    emit Harvested();
}
```
