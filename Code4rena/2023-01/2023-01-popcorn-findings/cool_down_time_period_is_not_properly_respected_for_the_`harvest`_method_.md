## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [cool down time period is not properly respected for the `harvest` method ](https://github.com/code-423n4/2023-01-popcorn-findings/issues/558) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L86
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L438-L450


# Vulnerability details


Harvest method is called on every deposit or withdraw into the `Vault` which further calls into the provided strategy.
This calling into strategy is limited by the cool down period. But in the current implementation is not properly respected.

## Impact
Setting the cool down period for a strategy harvest callback method is not working properly so that on every deposit/withdraw into `Vault` also the strategy is called every time.

## Proof of Concept
The main problem is that `lastHarvest` state variable is only set in the constructor 

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L86

and is not updated on strategy harvest method execution in the following lines:

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L438-L450

## Tools Used
Manual review

## Recommended Mitigation Steps
For the cool down period to work correctly update tha `lastHarvset` state variable like this:

```diff
    function harvest() public takeFees {
        if (
            address(strategy) != address(0) &&
            ((lastHarvest + harvestCooldown) < block.timestamp)
        ) {
+
+           lastHarvest = block.timestamp;
+
            // solhint-disable
            address(strategy).delegatecall(
                abi.encodeWithSignature("harvest()")
            );
        }

        emit Harvested();
    }
```