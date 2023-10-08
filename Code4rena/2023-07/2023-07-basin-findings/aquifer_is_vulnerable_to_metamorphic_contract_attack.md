## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-09

# [Aquifer is vulnerable to Metamorphic Contract Attack](https://github.com/code-423n4/2023-07-basin-findings/issues/168) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Aquifer.sol#L40-L52


# Vulnerability details

The [Aquifer](https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Aquifer.sol#L40-L64) contract supports multiple ways to deploy the `Well` contracts. More specifically, it supports `create` and `create2` at the same time. However, such a feature is vulnerable to the [Metamorphic Contract Attack](https://github.com/0age/metamorphic/tree/master). That is to say, attackers are capable to deploy two different `Well` implementations in the same address, which is recorded by `mapping(address => address) wellImplementations;`.

Although the Aquifer contract is claimed to be permissionless, it should not break the immutability. Thus, we consider it a medium-risk bug.

## Impact

The real implementation of the `Well` contract listed in `Aquifer` may be inconsistent with the expectation of users. Even worse, users may suffer from unexpected loss due to the change of contract logic.

## Proof of Concept

```
// the Aquifer contract
function boreWell(
    address implementation,
    bytes calldata immutableData,
    bytes calldata initFunctionCall,
    bytes32 salt
) external nonReentrant returns (address well) {
    if (immutableData.length > 0) {
        if (salt != bytes32(0)) {
            well = implementation.cloneDeterministic(immutableData, salt);
        } else {
            well = implementation.clone(immutableData);
        }
    } else {
        if (salt != bytes32(0)) {
            well = implementation.cloneDeterministic(salt);
        } else {
            well = implementation.clone();
        }
    }
    ...
}

// the cloneDeterministic() function
function cloneDeterministic(address implementation, bytes32 salt)
        internal
        returns (address instance)
    {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x21, 0x5af43d3d93803e602a57fd5bf3)
            mstore(0x14, implementation)
            mstore(0x00, 0x602c3d8160093d39f33d3d3d3d363d3d37363d73)
            instance := create2(0, 0x0c, 0x35, salt)
            // Restore the part of the free memory pointer that has been overwritten.
            mstore(0x21, 0)
            // If `instance` is zero, revert.
            if iszero(instance) {
                // Store the function selector of `DeploymentFailed()`.
                mstore(0x00, 0x30116425)
                // Revert with (offset, size).
                revert(0x1c, 0x04)
            }
        }
    }
```

As shown in the above code, attackers are capable to deploy new `Well` contracts through `cloneDeterministic` multiple times with the same input parameter `implementation`. And the `cloneDeterministic` function utilizes the following bytecode to deploy a new `Well` contract: `0x602c3d8160093d39f33d3d3d3d363d3d37363d73 + implementation + 5af43d3d93803e602a57fd5bf3`. That is to say, if the address (i.e., `implementation`) remains the same, then the address of the deployed `Well` contract also remains the same.

Normally, EVM would revert if anyone re-deploy a contract to the same address. However, if the `implementation` contract contains self-destruct logic, then attackers can re-deploy a new contract with different bytecode to the same address through `cloneDeterministic`.

Here is how we attack:
+ Assuming Bob deploys `Well_Implementation1` to address 1.
+ Bob invoke `Aquifer:boreWell` with address 1 as the parameter to get a newly deployed `Well1` contract at address 2.
+ Bob invokes the self-destruct logic in the `Well_Implementation1` contract and re-deploy a new contract to address 1 through [Metamorphic Contract](https://github.com/0age/metamorphic/tree/master), namely `Well_Implementation2`.
+ Bob invoke `Aquifer:boreWell` with address 1 again. Since the input of `create2` remains the same, a new contract is deployed to address 2 with new logic from `Well_Implementation2`.

## Recommended Mitigation Steps

Remove the `cloneDeterministic` feature, leaving the `clone` functionality only.


## Assessed type

Other