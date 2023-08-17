## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [`updatePoolAddress` functions always reverts when updating existing poolId](https://github.com/code-423n4/2023-06-stader-findings/issues/341) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/PoolUtils.sol#L55-L65


# Vulnerability details


## Impact

The purpose of the `updatePoolAddress` function is to update the pool address associated with an existing poolId. However, due to its internal invocation of the `verifyNewPool` function, the `updatePoolAddress` function always reverts, this occurs because the `verifyNewPool` function itself reverts when the specified poolId already exists. Consequently, it is not possible to update the pool address for an existing poolId.

## Proof of Concept

The issue occurs in the `updatePoolAddress` function below : 

File: PoolUtils.sol [Line 55-65](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/PoolUtils.sol#L55-L65)

```solidity
function updatePoolAddress(
    uint8 _poolId,
    address _newPoolAddress
) external override onlyExistingPoolId(_poolId) onlyRole(DEFAULT_ADMIN_ROLE) {
    UtilLib.checkNonZeroAddress(_newPoolAddress);
    // @audit always revert on exsiting poolId
    verifyNewPool(_poolId, _newPoolAddress); 
    poolAddressById[_poolId] = _newPoolAddress;
    emit PoolAddressUpdated(_poolId, _newPoolAddress);
}
```

As it can be seen from the code above, the `updatePoolAddress` function contains the `onlyExistingPoolId` modifier which means it can only be called for updating the pool address of an already exiting poolId. 

Before updating the pool address the `updatePoolAddress` function calls the `verifyNewPool` function below :
```solidity
function verifyNewPool(uint8 _poolId, address _poolAddress) internal view {
    if (
        INodeRegistry(IStaderPoolBase(_poolAddress).getNodeRegistry()).POOL_ID() != _poolId ||
        isExistingPoolId(_poolId)
    ) {
        revert ExistingOrMismatchingPoolId();
    }
}
```

It's clear that The function reverts when the poolId already exists meaning `isExistingPoolId(_poolId) == true`.

So to summarize the `updatePoolAddress` function reverts when the poolId does not exists and the `verifyNewPool` function reverts when the poolId exists, the two functions work on opposite conditions which means that when the `verifyNewPool` function is called inside the `updatePoolAddress` function it will automatically revert and hence the pool address of already existing poolId can never be updated.

## Tools Used

Manual review

## Recommended Mitigation Steps

Remove the `verifyNewPool` call inside the `updatePoolAddress` function and replace it with the following :

```solidity
function updatePoolAddress(
    uint8 _poolId,
    address _newPoolAddress
) external override onlyExistingPoolId(_poolId) onlyRole(DEFAULT_ADMIN_ROLE) {
    UtilLib.checkNonZeroAddress(_newPoolAddress);
    // @audit revert only when mismatch in poolId
    if (INodeRegistry(IStaderPoolBase(_poolAddress).getNodeRegistry()).POOL_ID() != _poolId) {
        revert MismatchingPoolId();
    }
    poolAddressById[_poolId] = _newPoolAddress;
    emit PoolAddressUpdated(_poolId, _newPoolAddress);
}
```


## Assessed type

Error