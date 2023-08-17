## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [`VaultProxy` implementation can be initialized by anyone and self-destructed](https://github.com/code-423n4/2023-06-stader-findings/issues/418) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/VaultProxy.sol#L20-L36
https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/VaultProxy.sol#L41-L50


# Vulnerability details

## Impact
When the `VaultFactory` contract is deployed and initialized, the `initialise` method on the newly created `VaultProxy` implementation contract is never called. As such, anyone can call that method and pass in whatever values they want as arguments. One important argument is the `_staderConfig` address, which controls where the `fallback` function will direct `delegatecall` operations. If an attacker passes in a contract that calls `selfdestruct` it will be run in the context of the `VaultProxy` implementation contract, and will erase all code from that address. Since the clones from the VaultProxy contract merely delegate calls to the implementation address, all subsequent calls for all created vaults from that implementation, will be treated like an EOA and return `true` even though calls to functions on that proxy were never executed.

## Proof of Concept
- First, an attacker deploys a contract called `AttackContract` that calls `selfdestruct` in its `fallback` function.
```
contract AttackContract {
    function getValidatorWithdrawalVaultImplementation() public view returns(address) {
        return address(this);
    }
    function getNodeELRewardVaultImplementation() public view returns(address) {
	return address(this);
    }
    fallback(bytes calldata _input) external payable returns(bytes memory _output) {
	selfdestruct(address(0));
    }
}
```
- The attacker calls the `initialise` method on the `VaultProxy` implementation contract. That address is stored in the `vaultProxyImplementation` variable on the `VaultFactory` contract. The attacker passes in the address of `AttackContract` as the `_staderConfig` argument for the `initialise` function.
- The attacker then calls a non-existent function on the `VaultProxy` implementation contract, which triggers it's `fallback` function. The `fallback` function calls `staderConfig.getNodeELRewardVaultImplementation()`, and since `staderConfig` is set the `AttackContract` address, it returns the address of the `AttackContract`. `delegatecall` runs the fallback function of `AttackContract` in its own execution environment. `selfdestruct` is called in the execution environment of the `VaultProxy` implementation, which erases the code at that address.
- All cloned copies of the `VaultProxy` implementation contract are now forwarding calls to an implementation address that has no code stored at it. These calls will be treated like calls to an EOA and return `true` for `success`.

## Tools Used
Manual Analysis

## Recommended Mitigation Steps
Prevent the `initialise` function from being called on the `VaultProxy` implementation contract by inheriting from OpenZeppelin's `Initializable` contract, like the system is doing in other contracts. Call the `_disableInitializers` function in the constructor and protect `initialise` with the `initializer` modifier. Alternatively, the `initialise` function can be called from the `initialize` function of the `VaultFactory` contract when the `VaultProxy` contract is instantiated.


## Assessed type

Access Control