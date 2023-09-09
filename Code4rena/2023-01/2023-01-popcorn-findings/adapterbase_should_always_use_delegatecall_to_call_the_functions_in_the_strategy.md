## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-21

# [AdapterBase should always use delegatecall to call the functions in the strategy](https://github.com/code-423n4/2023-01-popcorn-findings/issues/435) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L479-L483


# Vulnerability details

## Impact
The strategy contract will generally let the Adapter contract use delegatecall to call its functions
So IAdapter(address(this)).call is used frequently in strategy contracts, because when the Adapter calls the strategy's functions using delegatecall, address(this) is the Adapter
```solidity
  function harvest() public override {
    address router = abi.decode(IAdapter(address(this)).strategyConfig(), (address));
    address asset = IAdapter(address(this)).asset();
    ...
```
But in AdapterBase._verifyAndSetupStrategy, the verifyAdapterSelectorCompatibility/verifyAdapterCompatibility/setUp functions are not called with delegatecall, which causes the context of these functions to be the strategy contract
```solidity
    function _verifyAndSetupStrategy(bytes4[8] memory requiredSigs) internal {
        strategy.verifyAdapterSelectorCompatibility(requiredSigs);
        strategy.verifyAdapterCompatibility(strategyConfig);
        strategy.setUp(strategyConfig);
    }
```
and since the strategy contract does not implement the interface of the Adapter contract, these functions will fail, making it impossible to create a Vault using that strategy.
```solidity
  function verifyAdapterCompatibility(bytes memory data) public override {
    address router = abi.decode(data, (address));
    address asset = IAdapter(address(this)).asset();
```
More dangerously, if functions such as setup are executed successfully because they do not call the Adapter's functions, they may later error out when calling the havest function because the settings in setup are invalid
## Proof of Concept
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/abstracts/AdapterBase.sol#L479-L483
## Tools Used
None
## Recommended Mitigation Steps
In AdapterBase._verifyAndSetupStrategy, the verifyAdapterSelectorCompatibility/verifyAdapterCompatibility/setUp functions are called using delegatecall